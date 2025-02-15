<template>
  <div>
    <Portal to="navigation">
      <HeaderNavLink to="/server-management" name="Server Management"></HeaderNavLink>
      <HeaderNavLink
        to="/server-management/active-users"
        name="Active Users"
      ></HeaderNavLink>
    </Portal>

    <div class="flex justify-between items-center mb-8">
      <h1 class="h4 font-bold">Active Users</h1>
      <FormButton :icon-left="UserPlusIcon" @click="toggleInviteDialog">
        Invite
      </FormButton>
    </div>

    <FormTextInput
      size="lg"
      name="search"
      :custom-icon="MagnifyingGlassIcon"
      color="foundation"
      full-width
      search
      :show-clear="!!searchString"
      placeholder="Search Users"
      class="rounded-md border border-outline-3"
      @update:model-value="debounceSearchUpdate"
      @change="($event) => searchUpdateHandler($event.value)"
    />

    <ServerManagementTable
      class="mt-8"
      :headers="[
        { id: 'name', title: 'Name' },
        { id: 'email', title: 'Email' },
        { id: 'emailState', title: 'Email State' },
        { id: 'company', title: 'Company' },
        { id: 'role', title: 'Role' }
      ]"
      :items="users"
      :buttons="[{ icon: TrashIcon, label: 'Delete', action: openUserDeleteDialog }]"
      :column-classes="{
        name: 'col-span-3 truncate',
        email: 'col-span-3 truncate',
        emailState: 'col-span-2',
        company: 'col-span-2 truncate',
        role: 'col-span-2'
      }"
      :overflow-cells="true"
    >
      <template #name="{ item }">
        <div class="flex items-center gap-2">
          <UserAvatar v-if="isUser(item)" :user="item" />
          <span class="truncate">
            {{ isUser(item) ? item.name : '' }}
          </span>
        </div>
      </template>

      <template #email="{ item }">
        {{ isUser(item) ? item.email : '' }}
      </template>

      <template #emailState="{ item }">
        <div class="flex items-center gap-2 select-none">
          <template v-if="isUser(item) && item.verified">
            <ShieldCheckIcon class="h-4 w-4 text-primary" />
            <span>verified</span>
          </template>
          <template v-else>
            <ShieldExclamationIcon class="h-4 w-4 text-danger" />
            <span>not verified</span>
          </template>
        </div>
      </template>

      <template #company="{ item }">
        {{ isUser(item) ? item.company : '' }}
      </template>

      <template #role="{ item }">
        <FormSelectServerRoles
          :allow-guest="isGuestMode"
          allow-admin
          allow-archived
          :model-value="isUser(item) ? item.role : undefined"
          :disabled="isUser(item) && isCurrentUser(item)"
          fully-control-value
          @update:model-value="(newRoleValue) => isUser(item) && !isArray(newRoleValue) && newRoleValue && openChangeUserRoleDialog(item, newRoleValue as ServerRoles)"
        />
      </template>
    </ServerManagementTable>

    <CommonLoadingBar v-if="loading && !users?.length" loading />

    <InfiniteLoading
      v-if="users?.length"
      :settings="{ identifier: infiniteLoaderId }"
      class="-mt-24 -mb-24"
      @infinite="infiniteLoad"
    />

    <ServerManagementDeleteUserDialog
      v-model:open="showUserDeleteDialog"
      :user="userToModify"
      :result-variables="resultVariables"
    />

    <ServerManagementChangeUserRoleDialog
      v-model:open="showChangeUserRoleDialog"
      :user="userToModify"
      :old-role="oldRole"
      :new-role="newRole"
      hide-closer
    />
    <ServerManagementInviteDialog v-model:open="showInviteDialog" />
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import { useQuery } from '@vue/apollo-composable'
import { debounce, isArray } from 'lodash-es'
import { useLogger } from '~~/composables/logging'
import { InfiniteLoaderState } from '~~/lib/global/helpers/components'
import { Nullable, ServerRoles, Optional } from '@speckle/shared'
import { getUsersQuery } from '~~/lib/server-management/graphql/queries'
import { ItemType, UserItem } from '~~/lib/server-management/helpers/types'
import { isUser } from '~~/lib/server-management/helpers/utils'
import { useActiveUser } from '~~/lib/auth/composables/activeUser'
import {
  MagnifyingGlassIcon,
  ShieldExclamationIcon,
  ShieldCheckIcon,
  TrashIcon,
  UserPlusIcon
} from '@heroicons/vue/20/solid'
import { useServerInfo } from '~~/lib/core/composables/server'

definePageMeta({
  middleware: ['admin']
})

const logger = useLogger()
const { activeUser } = useActiveUser()
const { isGuestMode } = useServerInfo()

const userToModify: Ref<Nullable<UserItem>> = ref(null)
const searchString = ref('')
const showUserDeleteDialog = ref(false)
const showChangeUserRoleDialog = ref(false)
const newRole = ref<ServerRoles>()
const infiniteLoaderId = ref('')
const showInviteDialog = ref(false)

const queryVariables = computed(() => ({
  limit: 50,
  query: searchString.value
}))

const {
  result: extraPagesResult,
  fetchMore: fetchMorePages,
  variables: resultVariables,
  onResult,
  loading
} = useQuery(getUsersQuery, queryVariables)

const oldRole = computed(() => userToModify.value?.role as Optional<ServerRoles>)

const moreToLoad = computed(
  () =>
    !extraPagesResult.value?.admin?.userList ||
    extraPagesResult.value.admin.userList.items.length <
      extraPagesResult.value.admin.userList.totalCount
)

const users = computed(() => extraPagesResult.value?.admin.userList.items || [])

const isCurrentUser = (userItem: UserItem) => {
  return userItem.id === activeUser.value?.id
}

const openUserDeleteDialog = (item: ItemType) => {
  if (isUser(item)) {
    userToModify.value = item
    showUserDeleteDialog.value = true
  }
}

const openChangeUserRoleDialog = (user: UserItem, newRoleValue: ServerRoles) => {
  if (user.role === newRoleValue) {
    return
  }
  userToModify.value = user
  newRole.value = newRoleValue
  showChangeUserRoleDialog.value = true
}

const infiniteLoad = async (state: InfiniteLoaderState) => {
  const cursor = extraPagesResult.value?.admin?.userList.cursor || null
  if (!moreToLoad.value || !cursor) return state.complete()

  try {
    await fetchMorePages({
      variables: {
        cursor
      }
    })
  } catch (e) {
    logger.error(e)
    state.error()
    return
  }

  state.loaded()
  if (!moreToLoad.value) {
    state.complete()
  }
}

const searchUpdateHandler = (value: string) => {
  searchString.value = value
}

const debounceSearchUpdate = debounce(searchUpdateHandler, 500)

const calculateLoaderId = () => {
  infiniteLoaderId.value = resultVariables.value?.query || ''
}

const toggleInviteDialog = () => {
  showInviteDialog.value = true
}

onResult(calculateLoaderId)
</script>
