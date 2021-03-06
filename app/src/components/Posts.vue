<template>
    <section class="content">
        <p-header
            v-if="hasPosts"
            title="Posts">
            <header-search
                slot="search"
                ref="search"
                placeholder="Filter or search posts..."
                onChangeEventName="posts-filter-value-changed" />

            <btn-dropdown
                slot="buttons"
                buttonColor="green"
                localStorageKey="publii-current-editor"
                :previewIcon="true"
                :items="dropdownItems"
                defaultValue="tinymce" />
        </p-header>

        <ul
            v-if="dataLoaded && hasPosts"
            class="filters">
            <li
                :class="filterCssClasses('all')"
                @click="setFilter('')">
                All <span class="filter-count">({{ counters.all }})</span>
            </li>

            <li
                :class="filterCssClasses('published')"
                @click="setFilter('is:published')">
                Published <span class="filter-count">({{ counters.published }})</span>
            </li>

            <li
                v-if="counters.featured"
                :class="filterCssClasses('featured')"
                @click="setFilter('is:featured')">
                Featured <span class="filter-count">({{ counters.featured }})</span>
            </li>

            <li
                v-if="counters.hidden"
                :class="filterCssClasses('hidden')"
                @click="setFilter('is:hidden')">
                Hidden <span class="filter-count">({{ counters.hidden }})</span>
            </li>

            <li
                v-if="counters.excluded"
                :class="filterCssClasses('excluded')"
                @click="setFilter('is:excluded')">
                Excluded <span class="filter-count">({{ counters.excluded }})</span>
            </li>

            <li
                v-if="counters.drafts"
                :class="filterCssClasses('draft')"
                @click="setFilter('is:draft')">
                Drafts <span class="filter-count">({{ counters.drafts }})</span>
            </li>

            <li
                v-if="counters.trashed"
                :class="filterCssClasses('trashed')"
                @click="setFilter('is:trashed')">
                Trashed <span class="filter-count">({{counters.trashed }})</span>
            </li>
        </ul>

        <collection v-if="dataLoaded && !emptySearchResults && hasPosts">
            <collection-header slot="header">
                <collection-cell width="40px">
                    <checkbox
                        value="all"
                        :checked="anyCheckboxIsSelected"
                        :onClick="toggleAllCheckboxes.bind(this, false)"
                        @click.native="$bus.$emit('document-body-clicked')" />
                </collection-cell>

                <collection-cell :width="showModificationDateAsColumn ? 'calc(100% - 680px)' : 'calc(100% - 480px)'">
                    <span 
                        class="col-sortable-title"
                        @click="ordering('title')">
                        <template v-if="orderBy === 'title'">
                            <strong>Title</strong>
                        </template>
                        <template v-else>Title</template>

                        <span class="order-descending" v-if="orderBy === 'title' && order === 'ASC'"></span>
                        <span class="order-ascending" v-if="orderBy === 'title' && order === 'DESC'"></span>
                    </span>
                </collection-cell>

                <collection-cell width="200px">
                    <span 
                        class="col-sortable-title"
                        @click="ordering('created')">
                        <template v-if="orderBy === 'created'">
                            <strong>Publication date</strong>
                        </template>
                        <template v-else>Publication date</template>

                        <span class="order-descending" v-if="orderBy === 'created' && order === 'ASC'"></span>
                        <span class="order-ascending" v-if="orderBy === 'created' && order === 'DESC'"></span>
                    </span>
                </collection-cell>

                <collection-cell 
                    v-if="showModificationDateAsColumn"
                    width="200px">
                    <span 
                        class="col-sortable-title"
                        @click="ordering('modified')">
                        <template v-if="orderBy === 'modified'">
                            <strong>Modification date</strong>
                        </template>
                        <template v-else>Modification date</template>

                        <span class="order-descending" v-if="orderBy === 'modified' && order === 'ASC'"></span>
                        <span class="order-ascending" v-if="orderBy === 'modified' && order === 'DESC'"></span>
                    </span>
                </collection-cell>

                <collection-cell width="200px">
                    <span 
                        class="col-sortable-title"
                        @click="ordering('author')">
                        <template v-if="orderBy === 'author'">
                            <strong>Author</strong>
                        </template>
                        <template v-else>Author</template>

                        <span class="order-descending" v-if="orderBy === 'author' && order === 'ASC'"></span>
                        <span class="order-ascending" v-if="orderBy === 'author' && order === 'DESC'"></span>
                    </span>
                </collection-cell>

                <collection-cell width="40px">
                    <span 
                        class="col-sortable-title"
                        @click="ordering('id')">
                        <template v-if="orderBy === 'id'">
                            <strong>ID</strong>
                        </template>
                        <template v-else>ID</template>

                        <span class="order-descending" v-if="orderBy === 'id' && order === 'ASC'"></span>
                        <span class="order-ascending" v-if="orderBy === 'id' && order === 'DESC'"></span>
                    </span>
                </collection-cell>

                <div
                    v-if="anyCheckboxIsSelected"
                    class="tools">
                    <p-button
                        v-if="trashVisible"                        
                        icon="delete"
                        type="small light icon delete"
                        :onClick="bulkDelete">
                        Delete
                    </p-button>

                    <p-button
                        v-if="trashVisible"
                        icon="restore"
                        type="small light icon"
                        :onClick="bulkRestore">
                        Restore
                    </p-button>

                    <p-button
                        v-if="!trashVisible"
                        icon="trash"
                        type="small light icon delete"
                        :onClick="bulkTrash">
                        Move to trash
                    </p-button>

                    <p-button
                        v-if="!trashVisible"
                        icon="duplicate"
                        type="small light icon"
                        :onClick="bulkDuplicate">
                        Duplicate
                    </p-button>

                    <div 
                        v-if="!trashVisible"
                        class="dropdown-wrapper">
                        <p-button
                            icon="more"
                            :type="bulkDropdownVisible ? 'small light icon active' : 'small light icon'"
                            @click.native.stop="toggleBulkDropdown">
                            More
                        </p-button>

                        <ul 
                            v-if="bulkDropdownVisible"
                            class="dropdown">
                            <li 
                                v-if="selectedPostsNeedsStatus('published')"
                                @click="bulkPublish"> 
                                <icon                                
                                   size="xs"
                                   name="draft-post"
                                   primaryColor="color-8" />
                                Publish
                            </li>
                            <li 
                                v-if="selectedPostsNeedsStatus('draft')"
                                @click="bulkUnpublish">
                                <icon                                
                                   size="xs"
                                   name="draft-post"
                                   primaryColor="color-7" />
                                Mark as draft
                            </li>
                            <li 
                                v-if="selectedPostsNeedsStatus('featured')"
                                @click="bulkFeatured">
                                 <icon                                
                                   size="xs"
                                   name="featured-post"
                                   primaryColor="color-helper-6" />
                                Mark as featured
                            </li>
                            <li 
                                v-if="selectedPostsHaveStatus('featured')"
                                @click="bulkUnfeatured">
                                 <icon                                
                                   size="xs"
                                   name="featured-post"
                                   primaryColor="color-8" />
                                Mark as unfeatured
                            </li>
                            <li 
                                v-if="selectedPostsNeedsStatus('excluded_homepage')"
                                @click="bulkExclude">
                                <icon                                
                                   size="xs"
                                   name="excluded-post"
                                   primaryColor="color-3" />
                                Exclude from homepage
                            </li>
                            <li 
                                v-if="selectedPostsHaveStatus('excluded_homepage')"
                                @click="bulkInclude">
                                <icon                                
                                   size="xs"
                                   name="excluded-post"
                                   primaryColor="color-8" />
                                Include in homepage
                            </li>
                            <li 
                                v-if="selectedPostsNeedsStatus('hidden')"
                                @click="bulkHide">
                                <icon                                
                                   size="xs"
                                   name="hidden-post"
                                   primaryColor="color-7" />
                                Hide
                            </li>
                            <li 
                                v-if="selectedPostsHaveStatus('hidden')"
                                @click="bulkUnhide">
                                <icon                                
                                   size="xs"
                                   name="hidden-post"
                                   primaryColor="color-8" />
                                Unhide
                            </li>
                        </ul>
                    </div>
                </div>
            </collection-header>

            <collection-row
                v-for="(item, index) in items"
                slot="content"
                :data-is-draft="item.isDraft"
                :key="index">
                <collection-cell width="40px">
                    <checkbox
                        :value="item.id"
                        :checked="isChecked(item.id)"
                        :onClick="toggleSelection" />
                </collection-cell>

                <collection-cell
                    type="titles"
                    :width="showModificationDateAsColumn ? 'calc(100% - 680px)' : 'calc(100% - 480px)'">
                    <h2 class="title">
                        <a
                            href="#"
                            @click.prevent.stop="editPost(item.id, item.editor)">                           

                          {{ item.title }}

                            <icon
                                v-if="item.isFeatured"
                                size="xs"
                                name="featured-post"
                                primaryColor="color-helper-6"
                                title="This post is featured" />
                            <icon
                                v-if="item.isHidden"
                                size="xs"
                                name="hidden-post"
                                primaryColor="color-7"
                                title="This post is hidden" />
                            <icon
                                v-if="item.isExcludedOnHomepage"                               
                                name="excluded-post"
                                size="xs"
                                primaryColor="color-3"
                                title="This post is excluded from homepage" />
                            <icon
                                v-if="item.isDraft"
                                size="xs"
                                name="draft-post"
                                primaryColor="color-7"
                                title="This post is a draft" />
                        </a>
                    </h2>

                    <div 
                        v-if="showPostSlugs"
                        class="post-slug">
                        URL: /{{ item.slug }}
                    </div>

                    <div 
                        v-if="item.tags"
                        class="post-tags">
                        <a
                            v-for="tag in item.tags"
                            href="#"
                            :class="{ 'tag': true, 'is-main-tag': tag.id === item.mainTag }"
                            :key="'tag-' + tag.id"
                            @click.stop.prevent="setFilter('tag:' + tag.name)">
                            #{{ tag.name }}
                        </a>
                    </div>
                </collection-cell>

                <collection-cell
                    type="publish-dates"
                    width="200px">
                    <span class="publish-date">{{ getCreationDate(item.created) }}</span>
                    <span 
                        v-if="!showModificationDateAsColumn"
                        class="modify-date">
                        Last modified: {{ getModificationDate(item.modified) }}
                    </span>
                </collection-cell>

                <collection-cell
                    v-if="showModificationDateAsColumn"
                    type="modification-dates"
                    width="200px">
                    <span class="modify-date">
                        {{ getModificationDate(item.modified) }}
                    </span>
                </collection-cell>

                <collection-cell
                    type="authors"
                    width="200px">
                    <a
                        href="#"
                        @click.prevent.stop="setFilter('author:' + item.author)">
                        {{ item.author }}
                    </a>
                </collection-cell>

                <collection-cell
                    width="40px">
                    {{ item.id }}
                </collection-cell>
            </collection-row>
        </collection>

        <empty-state
            v-if="emptySearchResults"
            description="There are no posts matching your criteria."></empty-state>

        <div
            v-if="dataLoaded && !hasPosts"
            class="empty-state post">
            
           <div>               
                <img :src="'../src/assets/svg/' + $store.state.app.theme + '/wysiwyg-editor.svg'" height="286" width="331" />
                <h3>WYSIWYG editor</h3>
                <p>This editor provides a familiar word-processing experience, with additional tools for users that want to control every aspect of their page content.</p>
                <p-button
                    slot="button"
                    icon="add-site-mono"
                    type="icon"
                    :onClick="addNewPost.bind(this, 'tinymce')">
                    Add new post
                </p-button>    
           </div>
           
           <div>
                <img :src="'../src/assets/svg/' + $store.state.app.theme + '/block-editor.svg'" height="286" width="331" />
                <h3>Block editor</h3>
                <p>A modern and intuitive editor with shortkey and markdown support to make blogging easy, with no need to worry about HTML or other code elements.</p>
                <p-button
                    slot="button"
                    icon="add-site-mono"
                    type="icon"
                    :onClick="addNewPost.bind(this, 'blockeditor')">
                    Add new post 
                </p-button>    
           </div>
            
           <div>                
                <img :src="'../src/assets/svg/' + $store.state.app.theme + '/markdown-editor.svg'" height="286" width="331" />
                <h3>Markdown editor</h3>
                <p>This editor supports Markdown syntax as shorthand for producing content quickly; great for extensive, no-frills projects such as documentation. </p>
                <p-button
                    slot="button"
                    icon="add-site-mono"
                    type="icon"
                    :onClick="addNewPost.bind(this, 'markdown')">
                    Add new post
                </p-button>    
           </div>
        </div>
    </section>
</template>

<script>
import { ipcRenderer } from 'electron';
import CollectionCheckboxes from './mixins/CollectionCheckboxes.js';

export default {
    name: 'posts',
    mixins: [
        CollectionCheckboxes
    ],
    data () {
        return {
            bulkDropdownVisible: false,
            dataLoaded: false,
            filterValue: '',
            selectedItems: [],
            orderBy: 'id',
            order: 'DESC'
        };
    },
    computed: {
        items () {
            let items = this.$store.getters.sitePosts(this.filterValue, this.orderBy, this.order);

            items.forEach((item, i) => {
                if (item.tags.length) {
                    item.tags.sort((tagA, tagB) => tagA.name.localeCompare(tagB.name));
                }
            });

            return items;
        },
        hasPosts () {
            return this.$store.state.currentSite.posts && !!this.$store.state.currentSite.posts.length;
        },
        emptySearchResults () {
            return this.filterValue !== '' && !this.items.length;
        },
        trashVisible () {
            return this.filterValue.indexOf('is:trashed') > -1;
        },
        counters () {
            if(!this.$store.state.currentSite || !this.$store.state.currentSite.posts) {
                return {
                    all: 0,
                    published: 0,
                    featured: 0,
                    hidden: 0,
                    excluded: 0,
                    drafts: 0,
                    trashed: 0
                };
            }

            return {
                all: this.$store.state.currentSite.posts.filter((post) => post.status.indexOf('trashed') === -1).length,
                published: this.$store.state.currentSite.posts.filter((post) => post.status.indexOf('trashed') === -1 && post.status.indexOf('draft') === -1).length,
                featured: this.$store.state.currentSite.posts.filter((post) => post.status.indexOf('trashed') === -1 && post.status.indexOf('featured') > -1).length,
                hidden: this.$store.state.currentSite.posts.filter((post) => post.status.indexOf('trashed') === -1 && post.status.indexOf('hidden') > -1).length,
                excluded: this.$store.state.currentSite.posts.filter((post) => post.status.indexOf('trashed') === -1 && post.status.indexOf('excluded_homepage') > -1).length,
                drafts: this.$store.state.currentSite.posts.filter((post) => post.status.indexOf('trashed') === -1 && post.status.indexOf('draft') > -1).length,
                trashed: this.$store.state.currentSite.posts.filter((post) => post.status.indexOf('trashed') > -1).length
            }
        },
        showModificationDateAsColumn () {
            return this.$store.state.app.config.showModificationDateAsColumn;
        },
        dropdownItems () {
            return [
                {
                    label: 'Use WYSIWYG editor',
                    activeLabel: 'Add new post',
                    value: 'tinymce',
                    icon: 'wysiwyg',
                    isVisible: () => true,
                    onClick: this.addNewPost.bind(this, 'tinymce')
                },
                {
                    label: 'Use Block editor',
                    activeLabel: 'Add new post',
                    value: 'blockeditor',
                    icon: 'block',
                    isVisible: () => true,
                    onClick: this.addNewPost.bind(this, 'blockeditor')
                },
                {
                    label: 'Use Markdown editor',
                    activeLabel: 'Add new post',
                    value: 'markdown',
                    icon: 'markdown',
                    isVisible: () => true,
                    onClick: this.addNewPost.bind(this, 'markdown')
                }
            ]
        },
        showPostSlugs () {
            return this.$store.state.app.config.showPostSlugs;
        }
    },
    mounted () {
        this.orderBy = this.$store.state.ordering.posts.orderBy;
        this.order = this.$store.state.ordering.posts.order;
        this.$bus.$on('site-loaded', this.whenSiteLoaded);

        this.$bus.$on('posts-filter-value-changed', (newValue) => {
            this.filterValue = newValue.trim().toLowerCase();
        });

        this.$bus.$on('document-body-clicked', this.closeBulkDropdown);

        // It is available when user comes from Tags/Authors views
        let newFilterValue = localStorage.getItem('publii-posts-search-value');

        if(newFilterValue) {
            localStorage.removeItem('publii-posts-search-value');
            setTimeout (() => {
                this.setFilter(newFilterValue);
            }, 0);
        }

        this.$bus.$on('site-switched', () => {
            setTimeout(() => {
                this.saveOrdering(this.$store.state.ordering.posts.orderBy, this.$store.state.ordering.posts.order);
            }, 500);
        });

        this.$bus.$on('app-settings-saved', newSettings => {
            if (this.orderBy + ' ' + this.order !== newSettings.postsOrdering) {
                let order = newSettings.postsOrdering.split(' ');
                this.saveOrdering(order[0], order[1]);
            }
        });

        if (this.$store.state.currentSite.posts) {
            this.dataLoaded = true;
        }
    },
    methods: {
        addNewPost (editorType) {
            let siteName = this.$route.params.name;

            if(this.filterValue.trim() !== '' && this.$store.state.app.config.alwaysSaveSearchState) {
                localStorage.setItem('publii-posts-search-value', this.filterValue);
            }

            this.$store.commit('setEditorOpenState', true);
            this.$router.push('/site/' + siteName + '/posts/editor/' + editorType + '/');
        },
        editPost (id, editorType) {
            let siteName = this.$route.params.name;

            if(this.filterValue.trim() !== '') {
                localStorage.setItem('publii-posts-search-value', this.filterValue);
            }

            this.$store.commit('setEditorOpenState', true);
            this.$router.push('/site/' + siteName + '/posts/editor/' + editorType + '/' + id);

            return false;
        },
        setFilter (newValue) {
            if (this.$refs.search) {
                this.$refs.search.isOpen = newValue !== '';
                this.$refs.search.value = newValue;
                this.$refs.search.updateValue();
            }
        },
        filterCssClasses (type) {
            if(type !== 'all') {
                return {
                    'filter-value': true,
                    'filter-active': this.filterValue.indexOf('is:' + type) === 0
                };
            }

            return {
                'filter-value': true,
                'filter-active': this.filterValue.indexOf('is:') === -1
            };
        },
        getModificationDate (timestamp) {
            return this.$moment(timestamp).fromNow();
        },
        getCreationDate (timestamp) {
            if(this.$store.state.app.config.timeFormat == 12) {
                return this.$moment(timestamp).format('MMM DD, YYYY  hh:mm a');
            } else {
                return this.$moment(timestamp).format('MMM DD, YYYY  HH:mm');
            }
        },
        bulkDelete () {
            this.$bus.$emit('confirm-display', {
                message: 'Do you really want to remove selected posts? It cannot be undone.',
                okClick: this.deleteSelected
            });
        },
        deleteSelected () {
            let itemsToRemove = this.getSelectedItems();

            ipcRenderer.send('app-post-delete', {
                "site": this.$store.state.currentSite.config.name,
                "ids": itemsToRemove
            });

            ipcRenderer.once('app-post-deleted', () => {
                this.$store.commit('removePosts', itemsToRemove);
                this.selectedItems = [];

                this.$bus.$emit('message-display', {
                    message: 'Selected posts have been removed',
                    type: 'success',
                    lifeTime: 3
                });

                if (this.counters.trashed === 0) {
                    this.filterValue = '';
                }
            });
        },
        bulkTrash () {
            this.changeStateForSelected('trashed');
        },
        bulkPublish () {
            this.changeStateForSelected('published');
            this.changeStateForSelected('draft', true);
        },
        bulkUnpublish () {
            this.changeStateForSelected('published', true);
            this.changeStateForSelected('draft');
        },
        bulkFeatured () {
            this.changeStateForSelected('featured');
        },
        bulkUnfeatured () {
            this.changeStateForSelected('featured', true);
        },
        bulkExclude () {
            this.changeStateForSelected('excluded_homepage');
        },
        bulkInclude () {
            this.changeStateForSelected('excluded_homepage', true);
        },
        bulkHide () {
            this.changeStateForSelected('hidden');
        },
        bulkUnhide () {
            this.changeStateForSelected('hidden', true);
        },
        bulkDuplicate () {
            let itemsToDuplicate = this.getSelectedItems();

            ipcRenderer.send('app-post-duplicate', {
                "site": this.$store.state.currentSite.config.name,
                "ids": itemsToDuplicate
            });

            ipcRenderer.once('app-post-duplicated', (data) => {
                if(!data) {
                    this.$bus.$emit('message-display', {
                        message: 'An error occured during duplicating of selected posts. Please try again.',
                        type: 'warning',
                        lifeTime: 3
                    });

                    return;
                } else {
                    this.$bus.$emit('message-display', {
                        message: 'Selected posts have been duplicated',
                        type: 'success',
                        lifeTime: 3
                    });
                }

                this.selectedItems = [];

                ipcRenderer.send('app-site-reload', {
                    siteName: this.$store.state.currentSite.config.name
                });

                ipcRenderer.once('app-site-reloaded', (event, result) => {
                    this.$store.commit('setSiteConfig', result);
                    this.$store.commit('switchSite', result.data);
                });
            });
        },
        bulkRestore () {
            this.changeStateForSelected('trashed', true);
        },
        changeStateForSelected (status, inverse = false) {
            let itemsToChange = this.getSelectedItems();

            this.$store.commit('changePostsStatus', {
                postIDs: itemsToChange,
                status: status,
                inverse: inverse
            });

            ipcRenderer.send('app-post-status-change', {
                "site": this.$store.state.currentSite.config.name,
                "ids": itemsToChange,
                "status": status,
                "inverse": inverse
            });

            ipcRenderer.once('app-post-status-changed', () => {
                this.selectedItems = [];
            });

            this.$bus.$emit('message-display', {
                message: 'Status of the selected posts has been changed',
                type: 'success',
                lifeTime: 3
            });
        },
        whenSiteLoaded () {
            this.dataLoaded = true;

            setTimeout(() => {
                this.setFilter('');
            }, 0);
        },
        ordering (field) {
            if (field !== this.orderBy) {
                this.orderBy = field;
                this.order = 'DESC';
            } else {
                if (this.order === 'DESC') {
                    this.order = 'ASC';
                } else {
                    this.order = 'DESC';
                }
            }

            this.saveOrdering(this.orderBy, this.order);
        },
        saveOrdering (orderBy, order) {
            this.orderBy = orderBy;
            this.order = order;

            this.$store.commit('setOrdering', {
                type: 'posts',
                orderBy: this.orderBy,
                order: this.order
            });
        },
        toggleBulkDropdown () {
            this.bulkDropdownVisible = !this.bulkDropdownVisible;
        },
        closeBulkDropdown () {
            this.bulkDropdownVisible = false;
        },
        selectedPostsNeedsStatus (status) {
            let selectedPosts = this.items.filter(item => this.selectedItems.indexOf(item.id) > -1);

            if (!selectedPosts.length) {
                return false;
            }

            let postsWithoutGivenStatus = selectedPosts.filter(item => item.status.indexOf(status) === -1);

            return !!postsWithoutGivenStatus.length;
        },
        selectedPostsHaveStatus (status) {
            let selectedPosts = this.items.filter(item => this.selectedItems.indexOf(item.id) > -1);

            if (!selectedPosts.length) {
                return false;
            }

            let postsWithGivenStatus = selectedPosts.filter(item => item.status.indexOf(status) > -1);

            return !!postsWithGivenStatus.length;
        }
    },
    beforeDestroy () {
        this.$bus.$off('site-loaded', this.whenSiteLoaded);
        this.$bus.$off('posts-filter-value-changed');
        this.$bus.$off('document-body-clicked', this.closeBulkDropdown);
    }
}
</script>

<style lang="scss" scoped>
@import '../scss/variables.scss';
@import '../scss/empty-states.scss';

.header {
    .col {
        align-items: center;
        display: flex;

        .col-sortable-title {
            cursor: pointer;
        }
    }
}

.order-ascending,
.order-descending {
    margin-left: 3px;
    position: relative;
    &:after {
        border-top: solid 5px var(--icon-secondary-color);
        border-left: solid 5px transparent;
        border-right: solid 5px transparent;
        content: "";
        cursor: pointer;
        display: inline-block;
        height: 4px;
        left: 0;
        line-height: 1.1;
        opacity: 1;
        padding: 0;
        position: relative;
        text-align: center;
        top: 50%;
        transform: translateY(-50%);
        width: 8px;        
    }
}

.order-descending {
    &:after {
        border-top-color: transparent; 
        border-bottom: solid 5px var(--icon-secondary-color);                 
    }
}

.header {
    overflow-y: visible!important;
}

.item {
    .post-tags {
        display: flex;
        flex-wrap: wrap;

        a {
            order: 2;
            margin: .25rem .5rem .25rem 0;

            &.is-main-tag {
                order: 1;
            }
        }
    }

    .post-slug {
        color: var(--gray-4);
        font-size: 11px;
        margin-top: 5px;
    }
}

.filters {
    font-size: 1.4rem;
    list-style-type: none;
    margin: -2rem 0 0 0;
    padding: 0;
    position: relative;
    user-select: none;
    z-index: 1;

    .label {
        color: var(--text-light-color);
        float: left;
        margin-right: 1rem;
    }

    .filter-value {
        color: var(--text-light-color);
        cursor: pointer;
        display: inline-block;
        margin-right: 1rem;
        transition: var(--transition);

        &.filter-active {
            color: var(--link-primary-color);
            cursor: default;
        }

        &:hover {
            color: var(--link-primary-color);
        }

        &:last-child {
            border-right: none;
        }
    }
}

.tools {
   background: var(--bg-primary);
    display: flex;
    
    .button {
        padding-left: 4rem;
        position: relative;
        z-index: 0;
        
         &::before {
             content: "";
             background: var(--gray-1);  border-radius: 3px;
             display: block;
             left: -2px;
             opacity: 0;
             position: absolute;
             right: 0;           
             height: 100%;
             top: 0; 
             transition: all .15s cubic-bezier(0.4,0.0,0.2,1);
             transform: scale(.5);
             width: calc(100% + 2px);
             z-index: -1;
        }
        
        & + .button {
            margin: 0; 
            position: relative;
        }
        
        &:hover { 
            background: none;
            
            &::before {
                opacity: 1; 
                transform: scale(1);
            }
        }
    }
    
    .dropdown-wrapper {
        position: relative;

        .dropdown {
           background: var(--bg-secondary);
            border-radius: 3px;
            box-shadow: 0 2px 8px rgba(29, 39, 52, 0.15);
            left: 0;
            list-style-type: none;
            margin: 0;
            padding: 1rem 0;
            position: absolute;
            top: 4rem;
            width: auto;
            z-index: 1;

            li {
                color: var(--text-light-color);
                cursor: pointer;
                display: block;
                font-size: 1.4rem;
                font-weight: 500;
                padding: .8rem 2.4rem;
                white-space: nowrap;

                &:hover { 
                    background: var(--gray-1);
                    color: var(--text-primary-color);
                }
                
                & > svg {
                    margin-right: 4px;
                    vertical-align: text-bottom;
                }
            }
        }
    }
}
</style>
