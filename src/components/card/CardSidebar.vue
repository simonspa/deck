<!--
  - @copyright Copyright (c) 2018 Julius Härtl <jus@bitgrid.net>
  -
  - @author Julius Härtl <jus@bitgrid.net>
  -
  - @license GNU AGPL version 3 or any later version
  -
  - This program is free software: you can redistribute it and/or modify
  - it under the terms of the GNU Affero General Public License as
  - published by the Free Software Foundation, either version 3 of the
  - License, or (at your option) any later version.
  -
  - This program is distributed in the hope that it will be useful,
  - but WITHOUT ANY WARRANTY; without even the implied warranty of
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program. If not, see <http://www.gnu.org/licenses/>.
  -
  -->

<template>
	<AppSidebar v-if="currentBoard && currentCard && copiedCard"
		:title="currentCard.title"
		:subtitle="subtitle"
		@close="closeSidebar">
		<template #secondary-actions />
		<AppSidebarTab id="details"
			:order="0"
			:name="t('deck', 'Details')"
			icon="icon-home">
			<div class="section-wrapper">
				<div v-tooltip="t('deck', 'Tags')" class="section-label icon-tag">
					<span class="hidden-visually">{{ t('deck', 'Tags') }}</span>
				</div>
				<div class="section-details">
					<Multiselect v-model="allLabels"
						:multiple="true"
						:disabled="!canEdit"
						:options="currentBoard.labels"
						:placeholder="t('deck', 'Assign a tag to this card…')"
						:taggable="true"
						label="title"
						track-by="id"
						@select="addLabelToCard"
						@remove="removeLabelFromCard">
						<template #option="scope">
							<div :style="{ backgroundColor: '#' + scope.option.color, color: textColor(scope.option.color)}" class="tag">
								{{ scope.option.title }}
							</div>
						</template>
						<template #tag="scope">
							<div :style="{ backgroundColor: '#' + scope.option.color, color: textColor(scope.option.color)}" class="tag">
								{{ scope.option.title }}
							</div>
						</template>
					</Multiselect>
				</div>
			</div>

			<div class="section-wrapper">
				<div v-tooltip="t('deck', 'Assign to users')" class="section-label icon-group">
					<span class="hidden-visually">{{ t('deck', 'Assign to users/groups/circles') }}</span>
				</div>
				<div class="section-details">
					<Multiselect v-if="canEdit"
						v-model="assignedUsers"
						:multiple="true"
						:options="formatedAssignables"
						:user-select="true"
						:auto-limit="false"
						:placeholder="t('deck', 'Assign a user to this card…')"
						label="displayname"
						track-by="multiselectKey"
						@select="assignUserToCard"
						@remove="removeUserFromCard">
						<template #tag="scope">
							<div class="avatarlist--inline">
								<Avatar :user="scope.option.uid"
									:display-name="scope.option.displayname"
									:size="24"
									:is-no-user="scope.option.isNoUser"
									:disable-menu="true" />
							</div>
						</template>
					</Multiselect>
					<div v-else class="avatar-list--readonly">
						<Avatar v-for="option in currentCard.assignedUsers"
							:key="option.primaryKey"
							:user="option.participant.uid"
							:display-name="option.participant.displayname"
							:is-no-user="scope.option.isNoUser"
							:size="32" />
					</div>
				</div>
			</div>

			<div class="section-wrapper">
				<div v-tooltip="t('deck', 'Due date')" class="section-label icon-calendar-dark">
					<span class="hidden-visually">{{ t('deck', 'Due date') }}</span>
				</div>
				<div class="section-details">
					<DatetimePicker v-model="duedate"
						:placeholder="t('deck', 'Set a due date')"
						type="datetime"
						:disabled="saving || !canEdit"
						confirm />
					<Actions v-if="canEdit">
						<ActionButton v-if="copiedCard.duedate" icon="icon-delete" @click="removeDue()">
							{{ t('deck', 'Remove due date') }}
						</ActionButton>
					</Actions>
				</div>
			</div>

			<div class="section-wrapper">
				<CollectionList v-if="currentCard.id"
					:id="`${currentCard.id}`"
					:name="currentCard.title"
					type="deck-card" />
			</div>

			<h5>
				{{ t('deck', 'Description') }}
				<span v-if="copiedCard.descriptionLastEdit && !descriptionSaving">{{ t('deck', '(Unsaved)') }}</span>
				<span v-if="descriptionSaving">{{ t('deck', '(Saving…)') }}</span>
				<a v-tooltip="t('deck', 'Formatting help')"
					href="https://deck.readthedocs.io/en/latest/Markdown/"
					target="_blank"
					class="icon icon-info" />
				<Actions v-if="canEdit">
					<ActionButton v-if="descriptionEditing" icon="icon-attach" @click="showAttachmentModal()">
						{{ t('deck', 'Add Attachment') }}
					</ActionButton>
				</Actions>
				<Actions v-if="canEdit">
					<ActionButton v-if="!descriptionEditing" icon="icon-rename" @click="showEditor()">
						{{ t('deck', 'Edit description') }}
					</ActionButton>
					<ActionButton v-else icon="icon-toggle" @click="hideEditor()">
						{{ t('deck', 'View description') }}
					</ActionButton>
				</Actions>
			</h5>

			<div v-if="!descriptionEditing"
				id="description-preview"
				@click="clickedPreview"
				v-html="renderedDescription" />
			<VueEasymde v-else
				ref="markdownEditor"
				:value="copiedCard.description"
				:configs="mdeConfig"
				@input="updateDescription" />
		</AppSidebarTab>

		<AppSidebarTab id="attachments"
			:order="1"
			:name="t('deck', 'Attachments')"
			icon="icon-attach">
			<CardSidebarTabAttachments :card="currentCard" />
		</AppSidebarTab>

		<AppSidebarTab v-if="hasComments"
			id="comments"
			:order="2"
			:name="t('deck', 'Comments')"
			icon="icon-comment">
			<CardSidebarTabComments :card="currentCard" />
		</AppSidebarTab>

		<AppSidebarTab v-if="hasActivity"
			id="timeline"
			:order="3"
			:name="t('deck', 'Timeline')"
			icon="icon-activity">
			<CardSidebarTabActivity :card="currentCard" />
		</AppSidebarTab>
		<Modal v-if="modalShow" :title="t('deck', 'Choose attachment')" @close="modalShow=false">
			<div class="modal__content">
				<h3>{{ t('deck', 'Choose attachment') }}</h3>
				<AttachmentList
					:card-id="currentCard.id"
					:selectable="true"
					@selectAttachment="addAttachment" />
			</div>
		</Modal>
	</AppSidebar>
</template>

<script>
import { Avatar, Actions, ActionButton, Multiselect, AppSidebar, AppSidebarTab, DatetimePicker, Modal } from '@nextcloud/vue'
import { mapState, mapGetters } from 'vuex'
import Color from '../../mixins/color'
import { CollectionList } from 'nextcloud-vue-collections'
import CardSidebarTabAttachments from './CardSidebarTabAttachments'
import CardSidebarTabComments from './CardSidebarTabComments'
import CardSidebarTabActivity from './CardSidebarTabActivity'
import MarkdownIt from 'markdown-it'
import MarkdownItTaskLists from 'markdown-it-task-lists'
import { formatFileSize } from '@nextcloud/files'
import relativeDate from '../../mixins/relativeDate'
import AttachmentList from './AttachmentList'

const markdownIt = new MarkdownIt()
markdownIt.use(MarkdownItTaskLists, { enabled: true, label: true, labelAfter: true })

const capabilities = window.OC.getCapabilities()

export default {
	name: 'CardSidebar',
	components: {
		AppSidebar,
		AppSidebarTab,
		Multiselect,
		DatetimePicker,
		VueEasymde: () => import('vue-easymde/dist/VueEasyMDE.common'),
		Actions,
		ActionButton,
		Avatar,
		CollectionList,
		CardSidebarTabAttachments,
		CardSidebarTabComments,
		CardSidebarTabActivity,
		Modal,
		AttachmentList,
	},
	mixins: [Color, relativeDate],
	props: {
		id: {
			type: Number,
			required: true,
		},
	},
	data() {
		return {
			assignedUsers: null,
			addedLabelToCard: null,
			copiedCard: null,
			allLabels: null,

			saving: false,
			markdownIt: null,
			descriptionEditing: false,
			mdeConfig: {
				autoDownloadFontAwesome: false,
				spellChecker: false,
				autofocus: true,
				autosave: { enabled: false, uniqueId: 'unique' },
				toolbar: false,
			},
			lastModifiedRelative: null,
			lastCreatedRemative: null,
			descriptionSaveTimeout: null,
			descriptionSaving: false,
			hasActivity: capabilities && capabilities.activity,
			hasComments: !!OC.appswebroots['comments'],
			modalShow: false,
		}
	},
	computed: {
		...mapState({
			currentBoard: state => state.currentBoard,
		}),
		...mapGetters(['canEdit', 'assignables']),
		attachments() {
			return [...this.$store.getters.attachmentsByCard(this.id)].sort((a, b) => b.id - a.id)
		},
		mimetypeForAttachment() {
			return (mimetype) => {
				const url = OC.MimeType.getIconUrl(mimetype)
				const styles = {
					'background-image': `url("${url}")`,
				}
				return styles
			}
		},
		attachmentUrl() {
			return (attachment) => OC.generateUrl(`/apps/deck/cards/${attachment.cardId}/attachment/${attachment.id}`)
		},
		formattedFileSize() {
			return (filesize) => formatFileSize(filesize)
		},
		currentCard() {
			return this.$store.getters.cardById(this.id)
		},
		subtitle() {
			return t('deck', 'Modified') + ': ' + this.lastModifiedRelative + ' ' + t('deck', 'Created') + ': ' + this.lastCreatedRemative
		},
		formatedAssignables() {
			return this.assignables.map(item => {
				const assignable = {
					...item,
					user: item.primaryKey,
					displayName: item.displayname,
					icon: 'icon-user',
					isNoUser: false,
					multiselectKey: item.type + ':' + item.uid,
				}

				if (item.type === 1) {
					assignable.icon = 'icon-group'
					assignable.isNoUser = true
				}
				if (item.type === 7) {
					assignable.icon = 'icon-circles'
					assignable.isNoUser = true
				}

				return assignable
			})
		},
		duedate: {
			get() {
				return this.currentCard.duedate ? new Date(this.currentCard.duedate) : null
			},
			async set(val) {
				this.saving = true
				await this.$store.dispatch('updateCardDue', {
					...this.copiedCard,
					duedate: val ? (new Date(val)).toISOString() : null,
				})
				this.saving = false
			},
		},
		renderedDescription() {
			return markdownIt.render(this.copiedCard.description)
		},
	},
	watch: {
		currentCard() {
			this.initialize()
		},
	},
	created() {
		setInterval(this.updateRelativeTimestamps, 10000)
	},
	destroyed() {
		clearInterval(this.updateRelativeTimestamps)
	},
	mounted() {
		this.initialize()
	},
	methods: {
		initialize() {
			if (!this.currentCard) {
				return
			}

			this.copiedCard = JSON.parse(JSON.stringify(this.currentCard))
			this.allLabels = this.currentCard.labels

			if (this.currentCard.assignedUsers && this.currentCard.assignedUsers.length > 0) {
				this.assignedUsers = this.currentCard.assignedUsers.map((item) => ({
					...item.participant,
					isNoUser: item.participant.type !== 0,
					multiselectKey: item.participant.type + ':' + item.participant.primaryKey,
				}))
			} else {
				this.assignedUsers = []
			}

			this.desc = this.currentCard.description
			this.updateRelativeTimestamps()
		},
		showEditor() {
			if (!this.canEdit) {
				return
			}
			this.descriptionEditing = true
		},
		hideEditor() {
			this.descriptionEditing = false
		},
		showAttachmentModal() {
			this.modalShow = true
		},
		addAttachment(attachment) {
			const descString = this.$refs.markdownEditor.easymde.value()
			let embed = ''
			if (attachment.extendedData.mimetype.includes('image')) {
				embed = '!'
			}
			const attachmentString = embed + '[📎 ' + attachment.data + '](' + this.attachmentUrl(attachment) + ')'
			this.$refs.markdownEditor.easymde.value(descString + '\n' + attachmentString)
			this.modalShow = false
		},
		clickedPreview(e) {
			if (e.target.getAttribute('type') === 'checkbox') {
				const clickedIndex = [...document.querySelector('#description-preview').querySelectorAll('input')].findIndex((li) => li.id === e.target.id)
				const reg = /\[(X|\s|_|-)\]/ig
				let nth = 0
				const updatedDescription = this.copiedCard.description.replace(reg, (match, i, original) => {
					let result = match
					if ('' + nth++ === '' + clickedIndex) {
						if (match.match(/^\[\s\]/i)) {
							result = match.replace(/\[\s\]/i, '[x]')
						}
						if (match.match(/^\[x\]/i)) {
							result = match.replace(/\[x\]/i, '[ ]')
						}
						return result
					}
					return match
				})
				this.updateDescription(updatedDescription)
			}
		},
		updateRelativeTimestamps() {
			this.lastModifiedRelative = OC.Util.relativeModifiedDate(this.currentCard.lastModified * 1000)
			this.lastCreatedRemative = OC.Util.relativeModifiedDate(this.currentCard.createdAt * 1000)
		},
		setDue() {
			this.$store.dispatch('updateCardDue', this.copiedCard)
		},
		removeDue() {
			this.copiedCard.duedate = null
			this.$store.dispatch('updateCardDue', this.copiedCard)
		},
		updateDescription(text) {
			this.copiedCard.description = text
			this.copiedCard.descriptionLastEdit = Date.now()
			clearTimeout(this.descriptionSaveTimeout)
			this.descriptionSaveTimeout = setTimeout(async() => {
				if (!Object.prototype.hasOwnProperty.call(this.copiedCard, 'descriptionLastEdit') || this.descriptionSaving) {
					return
				}
				this.descriptionSaving = true
				await this.$store.dispatch('updateCardDesc', this.copiedCard)
				delete this.copiedCard.descriptionLastEdit
				this.descriptionSaving = false
			}, 2500)

		},

		closeSidebar() {
			this.$router.push({ name: 'board' })
		},

		assignUserToCard(user) {
			this.$store.dispatch('assignCardToUser', {
				card: this.copiedCard,
				assignee: {
					userId: user.uid,
					type: user.type,
				},
			})
		},

		removeUserFromCard(user) {
			this.$store.dispatch('removeUserFromCard', {
				card: this.copiedCard,
				assignee: {
					userId: user.uid,
					type: user.type,
				},
			})
		},

		addLabelToCard(newLabel) {
			this.copiedCard.labels.push(newLabel)
			const data = {
				card: this.copiedCard,
				labelId: newLabel.id,
			}
			this.$store.dispatch('addLabel', data)
		},

		removeLabelFromCard(removedLabel) {

			const removeIndex = this.copiedCard.labels.findIndex((label) => {
				return label.id === removedLabel.id
			})
			if (removeIndex !== -1) {
				this.copiedCard.labels.splice(removeIndex, 1)
			}

			const data = {
				card: this.copiedCard,
				labelId: removedLabel.id,
			}
			this.$store.dispatch('removeLabel', data)
		},
	},
}
</script>

<style>
	@import "~easymde/dist/easymde.min.css";
	.vue-easymde, .CodeMirror {
		border: none;
		margin: 0;
		padding: 0;
	}
	.editor-preview,
	.editor-statusbar {
		display: none;
	}

	#app-sidebar .app-sidebar-header__desc h4 {
		font-size: 12px !important;
	}
</style>

<style lang="scss" scoped>

	h5 {
		border-bottom: 1px solid var(--color-border);
		margin-top: 20px;
		margin-bottom: 5px;
		color: var(--color-text-maxcontrast);

		.icon-info {
			display: inline-block;
			width: 32px;
			height: 16px;
			float: right;
			opacity: .7;
		}

		.icon-attach {
			background-size: 16px;
			float: right;
			margin-top: -14px;
			opacity: .7;
		}

		.icon-toggle, .icon-rename {
			float: right;
			margin-top: -14px;
		}
	}

	aside::v-deep section {
		display: flex;
		flex-direction: column;
	}

	.section-wrapper {
		display: flex;
		max-width: 100%;
		margin-top: 10px;

		.section-label {
			background-position: 0px center;
			width: 28px;
			margin-left: 9px;
			flex-shrink: 0;
		}

		.section-details {
			flex-grow: 1;

			button.action-item--single {
				margin-top: -6px;
			}
		}
	}

	.tag {
		flex-grow: 0;
		flex-shrink: 1;
		overflow: hidden;
		padding: 1px 3px;
		border-radius: 3px;
		font-size: 85%;
		margin-right: 3px;
	}

	.avatarLabel {
		padding: 6px
	}

	.section-details::v-deep .multiselect__tags-wrap {
		flex-wrap: wrap;
	}

	.avatar-list--readonly .avatardiv {
		margin-right: 3px;
	}

	.avatarlist--inline {
		display: flex;
		align-items: center;
		margin-right: 3px;
		.avatarLabel {
			padding: 0;
		}
	}

	.multiselect::v-deep .multiselect__tags-wrap {
		z-index: 2;
	}

	.multiselect.multiselect--active::v-deep .multiselect__tags-wrap {
		z-index: 0;
	}

	#description-preview {
		min-height: 100px;

		&::v-deep input {
			min-height: auto;
		}
	}

	.modal__content {
		width: 25vw;
		min-width: 250px;
		height: 120px;
		text-align: center;
		margin: 20px 20px 60px 20px;
		padding-bottom: 20px;
	}

	.modal__content button {
		float: right;
		margin: 40px 3px 3px 0;
	}

</style>
