<template>
  <AppSidebar
    v-if="isActive"
    class="sidebar"
    :title="bookmark.title"
    :subtitle="bookmark.url"
    :background="background"
    @close="onClose"
  >
    <AppSidebarTab :name="t('bookmarks', 'Details')" icon="icon-info">
      <div>
        <h3>
          <span class="icon-calendar-dark" />
          {{ t('bookmarks', 'Creation date') }}
        </h3>
        {{ addedDate }}
      </div>
      <div>
        <h3><span class="icon-tag" /> {{ t('bookmarks', 'Tags') }}</h3>
        <Multiselect
          class="sidebar__tags"
          :value="tags"
          :auto-limit="false"
          :limit="7"
          :options="allTags"
          :multiple="true"
          :taggable="true"
          :placeholder="t('bookmarks', 'Select tags are create new ones')"
          @input="onTagsChange"
          @tag="onAddTag"
        />
      </div>
      <div>
        <h3><span class="icon-edit" /> {{ t('bookmarks', 'Notes') }}</h3>
        <div class="sidebar__notes" contenteditable @input="onNotesChange">
          {{ description }}
        </div>
      </div>
    </AppSidebarTab>
    <!--<AppSidebarTab :name="t('bookmarks', 'Sharing')" icon="icon-sharing" />-->
  </AppSidebar>
</template>
<script>
import AppSidebar from 'nextcloud-vue/dist/Components/AppSidebar';
import AppSidebarTab from 'nextcloud-vue/dist/Components/AppSidebarTab';
import Multiselect from 'nextcloud-vue/dist/Components/Multiselect';
import { generateUrl } from 'nextcloud-router';
import humanizeDuration from 'humanize-duration';
import { actions, mutations } from '../store/';

const MAX_RELATIVE_DATE = 1000 * 60 * 60 * 24 * 7; // one week

export default {
	name: 'SidebarBookmark',
	components: { AppSidebar, AppSidebarTab, Multiselect },
	data() {
		return {
			description: ''
		};
	},
	computed: {
		isActive() {
			if (!this.$store.state.sidebar) return false;
			return this.$store.state.sidebar.type === 'bookmark';
		},
		bookmark() {
			if (!this.isActive) return;
			return this.$store.getters.getBookmark(this.$store.state.sidebar.id);
		},
		background() {
			return generateUrl(`/apps/bookmarks/bookmark/${this.bookmark.id}/image`);
		},
		addedDate() {
			const date = new Date(Number(this.bookmark.added) * 1000);
			const age = Date.now() - date;
			if (age < MAX_RELATIVE_DATE) {
				const duration = humanizeDuration(age, {
					language: OC.getLanguage(),
					units: ['d', 'h', 'm', 's'],
					largest: 1
				});
				return this.t('bookmarks', '{time} ago', { time: duration });
			} else {
				return date.toLocaleDateString();
			}
		},
		tags() {
			return this.bookmark.tags;
		},
		allTags() {
			return this.$store.state.tags.map(tag => tag.name);
		}
	},
	watch: {
		bookmark(newBookmark) {
			this.description = newBookmark.description;
		}
	},
	created() {},
	methods: {
		onClose() {
			this.$store.commit(mutations.SET_SIDEBAR, null);
		},
		onNotesChange(e) {
			this.bookmark.description = e.target.textContent;
			this.scheduleSave();
		},
		onTagsChange(tags) {
			this.bookmark.tags = tags;
			this.scheduleSave();
		},
		onAddTag(tag) {
			this.bookmark.tags.push(tag);
			this.scheduleSave();
		},
		scheduleSave() {
			if (this.changeTimeout) clearTimeout(this.changeTimeout);
			this.changeTimeout = setTimeout(async() => {
				await this.$store.dispatch(actions.SAVE_BOOKMARK, this.bookmark.id);
				await this.$store.dispatch(actions.LOAD_TAGS);
			}, 1000);
		}
	}
};
</script>
<style>
.sidebar span[class^='icon-'] {
	display: inline-block;
	position: relative;
	top: 3px;
	opacity: 0.5;
}

.sidebar__tags {
	width: 100%;
}

.sidebar__notes {
	min-height: 200px !important;
	width: auto !important;
}
</style>
