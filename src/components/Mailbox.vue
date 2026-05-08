<!--
  - SPDX-FileCopyrightText: 2020 Nextcloud GmbH and Nextcloud contributors
  - SPDX-License-Identifier: AGPL-3.0-or-later
-->

<template>
	<div :class="{ 'empty-content': (!hasMessages && !loadingEnvelopes) || error }">
		<Error
			v-if="error"
			:error="t('mail', 'Could not open folder')"
			message=""
			role="alert" />
		<LoadingSkeleton v-else-if="loadingEnvelopes" :number-of-lines="20" />
		<Loading
			v-else-if="loadingCacheInitialization"
			:hint="t('mail', 'Loading messages …')"
			:slow-hint="t('mail', 'Indexing your messages. This can take a bit longer for larger folders.')" />
		<EmptyMailboxSection v-else-if="isPriorityInbox && !hasMessages" key="empty" />
		<EmptyMailbox v-else-if="!hasMessages" key="empty" />
		<div v-else>
			<NcCheckboxRadioSwitch
				:model-value="selectMode"
				:disabled="loadingAllMatching"
				type="checkbox"
				class="select-all-bar"
				@update:checked="selectMode ? unselectAll() : selectAll()">
				{{ selectAllLabel }}
			</NcCheckboxRadioSwitch>
			<div v-if="loadingAllMatching" class="select-all-loading">
				<NcLoadingIcon :size="16" />
				<span>{{ t('mail', 'Selecting messages…') }}</span>
			</div>
			<div v-if="selectAllHint && !loadingAllMatching && !allSelected" class="select-all-hint">
				{{ selectAllHint }}
			</div>
			<div
				v-if="allSelected && !selectAllMatching && flatEnvelopeList.length < totalEnvelopeCount"
				class="select-all-banner">
				<NcLoadingIcon v-if="loadingAllMatching" :size="16" />
				<span>{{ n('mail',
					'All {visible} message on this page selected. Select all {total} matching this filter?',
					'All {visible} messages on this page selected. Select all {total} matching this filter?',
					totalEnvelopeCount,
					{ visible: flatEnvelopeList.length, total: totalEnvelopeCount }) }}</span>
				<NcButton type="primary" @click="selectAllMatchingAction">
					{{ n('mail', 'Select all {count} message', 'Select all {count} messages', totalEnvelopeCount, { count: totalEnvelopeCount }) }}
				</NcButton>
			</div>
			<template v-if="hasGroupedEnvelopes && !isPriorityInbox">
			<div v-for="[label, group] in groupEnvelopes" :key="label">
				<SectionTitle class="section-title" :name="getLabelForGroup(label)" />
				<EnvelopeList
					:account="account"
					:mailbox="mailbox"
					:search-query="searchQuery"
					:envelopes="group"
					:loading-more="false"
					:load-more-button="false"
					:skip-transition="skipListTransition"
					:selection="selection"
					:flat-index="getGroupFlatIndex(label)"
					@delete="onDelete"
					@update:selection="onUpdateSelection"
					@select-range="onSelectRange" />
			</div>
		</template>
		<EnvelopeList
			v-else
			:account="account"
			:load-more-label="loadMoreLabel"
			:mailbox="mailbox"
			:search-query="searchQuery"
			:envelopes="envelopesToShow"
			:loading-more="loadingMore"
			:load-more-button="showLoadMore"
			:skip-transition="skipListTransition"
			:selection="selection"
			:flat-index="0"
			@delete="onDelete"
			@load-more="loadMore"
			@update:selection="onUpdateSelection"
			@select-range="onSelectRange" />
		</div>
	</div>
</template>

<script>
import { showError, showWarning } from '@nextcloud/dialogs'
import { mapStores } from 'pinia'
import { findIndex, propEq } from 'ramda'
import EmptyMailbox from './EmptyMailbox.vue'
import EmptyMailboxSection from './EmptyMailboxSection.vue'
import EnvelopeList from './EnvelopeList.vue'
import Error from './Error.vue'
import Loading from './Loading.vue'
import LoadingSkeleton from './LoadingSkeleton.vue'
import { NcButton, NcCheckboxRadioSwitch, NcLoadingIcon } from '@nextcloud/vue'
import SectionTitle from './SectionTitle.vue'
import MailboxLockedError from '../errors/MailboxLockedError.js'
import MailboxNotCachedError from '../errors/MailboxNotCachedError.js'
import { matchError } from '../errors/match.js'
import NoTrashMailboxConfiguredError
	from '../errors/NoTrashMailboxConfiguredError.js'
import logger from '../logger.js'
import useMainStore from '../store/mainStore.js'
import { mailboxHasRights } from '../util/acl.js'
import { wait } from '../util/wait.js'

export default {
	name: 'Mailbox',
	components: {
		EmptyMailboxSection,
		EmptyMailbox,
		EnvelopeList,
		Error,
		Loading,
		LoadingSkeleton,
		NcButton,
		NcCheckboxRadioSwitch,
		NcLoadingIcon,
		SectionTitle,
	},

	props: {
		groupEnvelopes: {
			type: Array,
			required: false,
			default: () => [],
		},

		loadMoreLabel: {
			type: String,
			default: undefined,
		},

		account: {
			type: Object,
			required: true,
		},

		mailbox: {
			type: Object,
			required: true,
		},

		bus: {
			type: Object,
			required: true,
		},

		paginate: {
			type: String,
			default: 'scroll',
		},

		initialPageSize: {
			type: Number,
			default: 20,
		},

		searchQuery: {
			type: String,
			required: false,
			default: undefined,
		},

		isPriorityInbox: {
			type: Boolean,
			required: false,
			default: false,
		},
	},

	data() {
		return {
			error: false,
			refreshing: false,
			loadingMore: false,
			loadingEnvelopes: false,
			loadingCacheInitialization: false,
			loadMailboxInterval: undefined,
			expanded: false,
			endReached: false,
			syncedMailboxes: new Set(),
			skipListTransition: false,
			selection: [],
			selectAllMatching: false,
			loadingAllMatching: false,
		}
	},

	computed: {
		...mapStores(useMainStore),
		sortOrder() {
			return this.mainStore.getPreference('sort-order', 'newest')
		},

		envelopes() {
			return this.mainStore.getEnvelopes(this.mailbox.databaseId, this.searchQuery)
		},

		envelopesToShow() {
			if (this.paginate === 'manual' && !this.expanded) {
				return this.envelopes.slice(0, this.initialPageSize)
			}
			return this.envelopes
		},

		hasGroupedEnvelopes() {
			return this.groupEnvelopes && this.groupEnvelopes.length > 0
		},

		hasMessages() {
			if (this.hasGroupedEnvelopes) {
				return this.groupEnvelopes.some(([, group]) => group.length > 0)
			}
			return this.envelopesToShow?.length > 0
		},

		showLoadMore() {
			return !this.endReached && this.paginate === 'manual'
		},

		/**
		 * Flat list of all visible envelopes, regardless of grouping.
		 * Used for shift-click range selection and Select All.
		 */
		flatEnvelopeList() {
			if (this.hasGroupedEnvelopes) {
				return this.groupEnvelopes.flatMap(([, group]) => group)
			}
			return this.envelopesToShow
		},

		/**
		 * Whether selection mode is active (at least one envelope selected).
		 */
		selectMode() {
			return this.selection.length > 0
		},

		/**
		 * Whether all visible envelopes are currently selected.
		 */
		allSelected() {
			return this.flatEnvelopeList.length > 0
				&& this.selection.length === this.flatEnvelopeList.length
		},

		/**
		 * Total count of envelopes matching the current filter.
		 * Falls back to visible count when total is unknown.
		 */
		totalEnvelopeCount() {
			// Use the loaded envelope count — a full count requires a backend API.
			// When endReached is true, we've loaded all matching envelopes.
			if (this.endReached) {
				return this.flatEnvelopeList.length
			}
			// Otherwise return loaded count; the banner handles the comparison.
			return this.envelopes.length
		},

		/**
		 * Context-aware label for the select-all checkbox.
		 * - With active filter: "Select {N} messages matching filter"
		 * - Without filter, more pages exist: "Select {N} messages on this page"
		 * - All loaded (no filter or all pages fetched): "Select all {N} messages"
		 */
		selectAllLabel() {
			const count = this.flatEnvelopeList.length
			const hasFilter = this.searchQuery && this.searchQuery.trim() !== 'match:allof'

			// When messages are selected, show the actual selection count
			if (this.selectMode) {
				return this.n('mail',
					'{count} selected',
					'{count} selected',
					this.selection.length, { count: this.selection.length })
			}

			if (hasFilter) {
				return this.n('mail',
					'Select {count} matching message',
					'Select {count} matching messages',
					count, { count })
			}
			if (!this.endReached && count > 0) {
				return this.n('mail',
					'Select {count} loaded message',
					'Select {count} loaded messages',
					count, { count })
			}
			return this.n('mail',
				'Select {count} message',
				'Select all {count} messages',
				count, { count })
		},

		/**
		 * Hint shown below the select-all checkbox to give context
		 * about how many messages are available and how to select more.
		 */
		selectAllHint() {
			if (!this.endReached && this.flatEnvelopeList.length > 0) {
				const hasFilter = this.searchQuery && this.searchQuery.trim() !== 'match:allof'
				if (hasFilter) {
					return this.t('mail', 'Scroll down to include more messages or click an avatar circle to select one at a time')
				}
				return this.t('mail', 'Scroll down to include more messages, use filter to refine, or click an avatar circle to select one at a time')
			}
			return ''
		},
	},

	watch: {
		mailbox() {
			this.selection = []
			this.loadEnvelopes()
				.then(() => {
					logger.debug(`syncing mailbox ${this.mailbox.databaseId} (${this.query}) after folder change`)
					this.sync(false)
				})
		},

		searchQuery() {
			this.selection = []
			this.selectAllMatching = false
			this.loadingAllMatching = false
			this.loadEnvelopes()
		},

		sortOrder() {
			this.selection = []
			this.loadEnvelopes()
		},
	},

	created() {
		this.bus.on('load-more', this.onScroll)
		this.bus.on('delete', this.onDelete)
		this.bus.on('archive', this.onArchive)
		this.bus.on('shortcut', this.handleShortcut)
		this.bus.on('select-all-matching', this.onBusSelectAllMatching)
		this.loadMailboxInterval = setInterval(this.loadMailbox, 60000)
	},

	async mounted() {
		if (this.mainStore.hasFetchedInitialEnvelopes) {
			return
		}

		await this.loadEnvelopes()
		logger.debug(`syncing folder ${this.mailbox.databaseId} (${this.searchQuery}) after mount`)
		await this.sync(false)

		await this.prefetchOtherMailboxes()

		this.mainStore.setHasFetchedInitialEnvelopesMutation(true)
	},

	unmounted() {
		this.bus.off('load-more', this.onScroll)
		this.bus.off('delete', this.onDelete)
		this.bus.off('archive', this.onArchive)
		this.bus.off('shortcut', this.handleShortcut)
		this.bus.off('select-all-matching', this.onBusSelectAllMatching)
		this.stopInterval()
	},

	methods: {
		initializeCache() {
			this.loadingCacheInitialization = true
			this.error = false

			logger.debug(`syncing folder ${this.mailbox.databaseId} (${this.query}) during cache initalization`)
			this.sync(true)
				.then(() => {
					this.loadingCacheInitialization = false

					return this.loadEnvelopes()
				})
		},

		async loadEnvelopes() {
			logger.debug(`Fetching envelopes for folder ${this.mailbox.databaseId} (${this.searchQuery})`, this.mailbox)
			if (!this.syncedMailboxes.has(this.mailbox.databaseId + (this.searchQuery ?? ''))) {
				// Only trigger skeleton if we didn't sync envelopes yet
				this.loadingEnvelopes = true
			} else {
				this.skipListTransition = true
				this.$nextTick(() => {
					this.skipListTransition = false
				})
			}

			this.loadingCacheInitialization = false
			this.error = false

			try {
				const envelopes = await this.mainStore.fetchEnvelopes({
					mailboxId: this.mailbox.databaseId,
					query: this.searchQuery,
					limit: this.initialPageSize,
				})

				logger.debug(envelopes.length + ' envelopes fetched', { envelopes })

				this.syncedMailboxes.add(this.mailbox.databaseId + (this.searchQuery ?? ''))
				this.loadingEnvelopes = false
			} catch (error) {
				await matchError(error, {
					[MailboxLockedError.getName()]: async (error) => {
						logger.info(`Mailbox ${this.mailbox.databaseId} (${this.searchQuery}) is locked`, { error })
						await wait(15 * 1000)
						// Keep trying
						await this.loadEnvelopes()
					},
					[MailboxNotCachedError.getName()]: async (error) => {
						logger.info(`Mailbox ${this.mailbox.databaseId} (${this.searchQuery}) not cached. Triggering initialization`, { error })
						this.loadingEnvelopes = false

						try {
							await this.initializeCache()
						} catch (error) {
							logger.error(`Could not initialize cache of folder ${this.mailbox.databaseId} (${this.searchQuery})`, { error })
							this.error = error
						}
					},
					default: (error) => {
						logger.error(`Could not fetch envelopes of folder ${this.mailbox.databaseId} (${this.searchQuery})`, { error })
						this.loadingEnvelopes = false
						this.error = error
					},
				})
			}
		},

		async loadMore() {
			if (!this.expanded && this.envelopesToShow.length < this.envelopes.length) {
				logger.debug('expanding envelope list')
				this.expanded = true
				return
			}

			logger.debug('fetching next envelope page')
			this.loadingMore = true

			try {
				const envelopes = await this.mainStore.fetchNextEnvelopePage({
					mailboxId: this.mailbox.databaseId,
					query: this.searchQuery,
				})
				if (envelopes.length === 0) {
					logger.info('envelope list end reached')
					this.endReached = true
				}
			} catch (error) {
				logger.error('could not fetch next envelope page', { error })
			} finally {
				this.loadingMore = false
			}
		},

		async prefetchOtherMailboxes() {
			for (const mailbox of this.mainStore.getRecursiveMailboxIterator(this.account.id)) {
				if (mailbox.databaseId === this.mailbox.databaseId) {
					continue
				}

				if (!mailbox.isSubscribed) {
					continue
				}

				try {
					const envelopes = await this.mainStore.fetchEnvelopes({
						mailboxId: mailbox.databaseId,
						limit: this.initialPageSize,
						includeCacheBuster: true,
					})
					this.syncedMailboxes.add(mailbox.databaseId + (this.searchQuery ?? ''))
					logger.debug(`Prefetched ${envelopes.length} envelopes for folder ${mailbox.displayName} (${mailbox.databaseId})`)
				} catch (error) {
					if (error instanceof MailboxNotCachedError) {
						// Just ignore
						continue
					}

					logger.error(`Failed to prefetch envelopes for folder ${mailbox.displayName} (${mailbox.databaseId}): ${error}`, {
						error,
					})
				}
			}
		},

		hasDeleteAcl() {
			return mailboxHasRights(this.mailbox, 'x')
		},

		hasSeenAcl() {
			return mailboxHasRights(this.mailbox, 's')
		},

		hasArchiveAcl() {
			return mailboxHasRights(this.mailbox, 'te')
		},

		async handleShortcut(e) {
			const envelopes = this.envelopes
			const currentId = parseInt(this.$route.params.threadId, 10)

			const env = envelopes.find((e) => e.databaseId === currentId)
			const idx = envelopes.indexOf(env)
			let next

			if (e.srcKey !== 'refresh' && !env) {
				logger.debug('envelope is not in the list, ignoring shortcut', {
					srcKey: e.srcKey,
				})
			}

			switch (e.srcKey) {
				case 'next':
				case 'prev':
					if (e.srcKey === 'next') {
						next = envelopes[idx + 1]
					} else {
						next = envelopes[idx - 1]
					}

					if (!next) {
						logger.debug('ignoring shortcut: head or tail of envelope list reached', {
							envelopes,
							idx,
							srcKey: e.srcKey,
						})
						return
					}

					// Keep the selected account-mailbox combination, but navigate to a different message
					// (it's not a bug that we don't use next.accountId and next.mailboxId here)
					this.$router.push({
						name: 'message',
						params: {
							mailboxId: this.$route.params.mailboxId,
							filter: this.$route.params.filter ? this.$route.params.filter : undefined,
							threadId: next.databaseId,
						},
					})
					break
				case 'del':
					if (!this.hasDeleteAcl()) {
						return
					}
					logger.debug('deleting', { env })
					this.onDelete(env.databaseId)
					try {
						await this.mainStore.deleteThread({
							envelope: env,
						})
					} catch (error) {
						logger.error('could not delete envelope', {
							env,
							error,
						})

						showError(await matchError(error, {
							[NoTrashMailboxConfiguredError.getName()]() {
								return t('mail', 'No trash folder configured')
							},
							default() {
								return t('mail', 'Could not delete message')
							},
						}))
					}

					break
				case 'arch':
					logger.debug('archiving via shortcut')

					if (this.account.archiveMailboxId === null) {
						showWarning(t('mail', 'To archive a message please configure an archive folder in account settings'))
						return
					}

					if (!this.hasArchiveAcl()) {
						showWarning(t('mail', 'You are not allowed to move this message to the archive folder and/or delete this message from the current folder'))
						return
					}

					if (env.mailboxId === this.account.archiveMailboxId) {
						logger.debug('message is already in archive folder')
						return
					}

					logger.debug('archiving', { env })
					this.onDelete(env.databaseId)
					try {
						await this.mainStore.moveThread({
							envelope: env,
							destMailboxId: this.account.archiveMailboxId,
						})
					} catch (error) {
						logger.error('could not archive envelope', {
							env,
							error,
						})

						showError(t('mail', 'Could not archive message'))
					}
					break
				case 'flag':
					logger.debug('flagging envelope via shortkey', { env })
					this.mainStore.toggleEnvelopeFlagged(env).catch((error) => logger.error('could not flag envelope via shortkey', {
						env,
						error,
					}))
					break
				case 'refresh':
					logger.debug(`syncing folder ${this.mailbox.databaseId} (${this.searchQuery}) per shortcut`)
					this.sync(false)

					break
				case 'unseen':
					logger.debug('marking as seen/unseen via shortcut')

					if (!this.hasSeenAcl()) {
						showWarning(t('mail', 'Your IMAP server does not support storing the seen/unseen state.'))
						return
					}

					logger.debug('marking as seen/unseen', { env })
					try {
						await this.mainStore.toggleEnvelopeSeen({ envelope: env })
					} catch (error) {
						logger.error('could not mark envelope as seen/unseen via shortkey', {
							env,
							error,
						})
						showError(t('mail', 'Could not mark message as seen/unseen'))
					}
					break
				default:
					logger.warn('shortcut ' + e.srcKey + ' is unknown. ignoring.')
			}
		},

		async sync(init = false) {
			if (this.refreshing) {
				logger.debug(`already sync'ing folder ${this.mailbox.databaseId} (${this.searchQuery}), aborting`, { init })
				return
			}

			this.refreshing = true
			try {
				await this.mainStore.syncEnvelopes({
					mailboxId: this.mailbox.databaseId,
					query: this.searchQuery,
					init,
				})
			} catch (error) {
				matchError(error, {
					[MailboxLockedError.getName()](error) {
						logger.info('Background sync failed because the folder is locked', {
							error,
							init,
						})
					},
					default(error) {
						logger.error('Could not sync envelopes: ' + error.message, {
							error,
							init,
						})
					},
				})
				throw error
			} finally {
				this.refreshing = false
				logger.debug(`finished sync'ing folder ${this.mailbox.databaseId} (${this.searchQuery})`, { init })

				this.mainStore.updateSyncTimestamp()
			}
		},

		// onDelete(id): Load more message and navigate to other message if needed
		// id: The id of the message being delete
		onDelete(id) {
			// Remove from selection if selected
			this.selection = this.selection.filter((selectedId) => selectedId !== id)

			// Get a new message
			this.mainStore.fetchNextEnvelopes({
				mailboxId: this.mailbox.databaseId,
				query: this.searchQuery,
				quantity: 1,
			})
			const idx = findIndex(propEq(id, 'databaseId'), this.envelopes)
			if (idx === -1) {
				logger.debug('envelope to delete does not exist in envelope list')
				return
			}
			if (id !== this.$route.params.threadId) {
				logger.debug('other message open, not jumping to the next/previous message')
				return
			}

			const next = this.envelopes[idx === 0 ? 1 : idx - 1]
			if (!next) {
				logger.debug('no next/previous envelope, not navigating')
				return
			}

			// Keep the selected mailbox, but navigate to a different message
			// (it's not a bug that we don't use next.mailboxId here)
			this.$router.push({
				name: 'message',
				params: {
					mailboxId: this.$route.params.mailboxId,
					filter: this.$route.params.filter ? this.$route.params.filter : undefined,
					threadId: next.databaseId,
				},
			})
		},

		onScroll() {
			if (this.paginate !== 'scroll') {
				logger.debug('ignoring scroll pagination')
				return
			}

			this.loadMore()
		},

		async loadMailbox() {
			// When the account is unified or inbox, return nothing, else sync the mailbox
			if (this.account.isUnified || this.mailbox.specialRole === 'inbox') {
				return
			}
			try {
				logger.debug(`syncing folder ${this.mailbox.databaseId} (${this.searchQuery}) in background`)
				await this.sync(false)
			} catch (error) {
				logger.error('Background sync failed: ' + error.message, { error })
			}
		},

		stopInterval() {
			clearInterval(this.loadMailboxInterval)
			this.loadMailboxInterval = undefined
		},

		/**
		 * Compute the flat index offset for a group label.
		 * Used by grouped EnvelopeList children to emit
		 * correct global indices for shift-click range selection.
		 *
		 * @param {string} label The group label key
		 * @return {number} Flat index of the first envelope in this group
		 */
		getGroupFlatIndex(label) {
			let offset = 0
			for (const [groupLabel, group] of this.groupEnvelopes) {
				if (groupLabel === label) {
					return offset
				}
				offset += group.length
			}
			return offset
		},

		/**
		 * Handle a child EnvelopeList updating its selection.
		 * The child emits its full new selection array (local IDs),
		 * and we merge it with the global selection, replacing
		 * any IDs from this child's visible envelope set.
		 *
		 * @param {number[]} childSelection Array of selected envelope databaseIds
		 * @param {object[]} childEnvelopes Array of envelopes visible in this child
		 */
		onUpdateSelection(childSelection, childEnvelopes) {
			const childIds = new Set(childEnvelopes.map((e) => e.databaseId))
			// Remove all IDs from this child's scope, then add the new selection
			this.selection = this.selection.filter((id) => !childIds.has(id))
			this.selection.push(...childSelection)
		},

		/**
		 * Handle shift-click range selection across the flat envelope list.
		 * Called by a child EnvelopeList with global flat indices.
		 *
		 * @param {number} fromIndex Start of the range (global flat index)
		 * @param {number} toIndex End of the range (global flat index)
		 * @param {boolean} deselect If true, remove the range from selection
		 */
		onSelectRange(fromIndex, toIndex, deselect = false) {
			const start = Math.min(fromIndex, toIndex)
			const end = Math.max(fromIndex, toIndex)
			const idsInRange = new Set(
				this.flatEnvelopeList
					.slice(start, end + 1)
					.map((e) => e.databaseId),
			)
			if (deselect) {
				this.selection = this.selection.filter((id) => !idsInRange.has(id))
			} else {
				const newSelection = new Set(this.selection)
				for (const id of idsInRange) {
					newSelection.add(id)
				}
				this.selection = [...newSelection]
			}
		},

		/**
		 * Select all visible envelopes.
		 */
		selectAll() {
			this.selection = this.flatEnvelopeList.map((e) => e.databaseId)
		},

		/**
		 * Select all messages matching the current filter across all pages.
		 * Loads additional pages of envelopes and adds them to the selection.
		 */
		async selectAllMatchingAction() {
			this.loadingAllMatching = true
			this.selectAllMatching = true

			try {
				// Load remaining pages until all envelopes are fetched
				while (!this.endReached) {
					await this.loadMore()
				}
				// Now select all loaded envelopes
				this.selection = this.flatEnvelopeList.map((e) => e.databaseId)
			} catch (error) {
				logger.error('Failed to load all matching envelopes', { error })
			} finally {
				this.loadingAllMatching = false
			}
		},

		/**
		 * Handler for the 'select-all-matching' bus event emitted from
		 * SearchMessages when the user clicks 'Select all matching'.
		 * Waits for envelopes to finish loading, then selects all.
		 */
		async onBusSelectAllMatching() {
			// Show spinner immediately
			this.loadingAllMatching = true
			this.selectAllMatching = true

			// Force a fresh load of the first page with the current query
			this.endReached = false
			this.syncedMailboxes.delete(this.mailbox.databaseId + (this.searchQuery ?? ''))
			await this.loadEnvelopes()

			// Now load remaining pages and select all
			await this.selectAllMatchingAction()
		},

		/**
		 * Clear the current selection.
		 */
		unselectAll() {
			this.selection = []
			this.selectAllMatching = false
			this.loadingAllMatching = false
		},

		getLabelForGroup(group) {
			switch (group) {
				case 'lastHour':
					return t('mail', 'Last hour')
				case 'today':
					return t('mail', 'Today')
				case 'yesterday':
					return t('mail', 'Yesterday')
				case 'lastWeek':
					return t('mail', 'Last week')
				case 'lastMonth':
					return t('mail', 'Last month')
				default:
					return group
			}
		},
	},
}
</script>

<style lang="scss" scoped>
// Fix vertical space between sections in priority inbox
.nameimportant {
	:deep(#load-more-mail-messages) {
		margin-top: 0;
	}
}

.empty-content {
	height: 100%;
	display: flex;
	justify-content: center;
}

.select-all-bar {
	display: flex;
	align-items: center;
	margin-top: 8px;
	padding: 4px 8px;
	cursor: pointer;
	border-bottom: 1px solid var(--color-border);
	&:hover {
		background-color: var(--color-background-hover);
	}
}

.select-all-loading {
	display: flex;
	align-items: center;
	gap: 8px;
	padding: 4px 8px 4px 36px;
	color: var(--color-text-maxcontrast);
	font-size: var(--default-font-size);
	border-bottom: 1px solid var(--color-border);
}

.select-all-hint {
	padding: 2px 8px 2px 36px;
	color: var(--color-text-maxcontrast);
	font-size: calc(var(--default-font-size) * 0.85);
	border-bottom: 1px solid var(--color-border);
}

.select-all-banner {
	display: flex;
	align-items: center;
	gap: 12px;
	padding: 8px 12px;
	background-color: var(--color-primary-light);
	border-bottom: 1px solid var(--color-border);
	font-size: var(--default-font-size);

	span {
		flex: 1;
	}
}
</style>
