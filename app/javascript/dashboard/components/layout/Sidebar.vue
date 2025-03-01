<template>
  <aside class="sidebar animated shrink columns">
    <div class="logo">
      <router-link :to="dashboardPath" replace>
        <img :src="globalConfig.logo" :alt="globalConfig.installationName" />
      </router-link>
    </div>

    <div class="main-nav">
      <transition-group name="menu-list" tag="ul" class="menu vertical">
        <sidebar-item
          v-for="item in accessibleMenuItems"
          :key="item.toState"
          :menu-item="item"
        />
        <sidebar-item
          v-if="shouldShowTeams"
          :key="teamSection.toState"
          :menu-item="teamSection"
        />
        <sidebar-item
          v-if="shouldShowSidebarItem"
          :key="inboxSection.toState"
          :menu-item="inboxSection"
        />
        <sidebar-item
          v-if="shouldShowSidebarItem"
          :key="labelSection.toState"
          :menu-item="labelSection"
          @add-label="showAddLabelPopup"
        />
        <sidebar-item
          v-if="showShowContactSideMenu"
          :key="contactLabelSection.key"
          :menu-item="contactLabelSection"
          @add-label="showAddLabelPopup"
        />
      </transition-group>
    </div>

    <div class="bottom-nav">
      <availability-status />
    </div>

    <div class="bottom-nav app-context-menu" @click="toggleOptions">
      <agent-details @show-options="toggleOptions" />
      <notification-bell />
      <span class="current-user--options icon ion-android-more-vertical" />
      <options-menu
        :show="showOptionsMenu"
        @toggle-accounts="toggleAccountModal"
        @show-support-chat-window="toggleSupportChatWindow"
        @key-shortcut-modal="toggleKeyShortcutModal"
        @close="toggleOptions"
      />
    </div>

    <woot-key-shortcut-modal
      v-if="showShortcutModal"
      @close="closeKeyShortcutModal"
      @clickaway="closeKeyShortcutModal"
    />

    <account-selector
      :show-account-modal="showAccountModal"
      @close-account-modal="toggleAccountModal"
      @show-create-account-modal="openCreateAccountModal"
    />

    <add-account-modal
      :show="showCreateAccountModal"
      @close-account-create-modal="closeCreateAccountModal"
    />

    <woot-modal :show.sync="showAddLabelModal" :on-close="hideAddLabelPopup">
      <add-label-modal @close="hideAddLabelPopup" />
    </woot-modal>
  </aside>
</template>

<script>
import { mapGetters } from 'vuex';

import adminMixin from '../../mixins/isAdmin';
import SidebarItem from './SidebarItem';
import AvailabilityStatus from './AvailabilityStatus';
import { frontendURL } from '../../helper/URLHelper';
import { getSidebarItems } from '../../i18n/default-sidebar';
import alertMixin from 'shared/mixins/alertMixin';
import NotificationBell from './sidebarComponents/NotificationBell';
import AgentDetails from './sidebarComponents/AgentDetails.vue';
import OptionsMenu from './sidebarComponents/OptionsMenu.vue';
import AccountSelector from './sidebarComponents/AccountSelector.vue';
import AddAccountModal from './sidebarComponents/AddAccountModal.vue';
import AddLabelModal from '../../routes/dashboard/settings/labels/AddLabel';
import WootKeyShortcutModal from 'components/widgets/modal/WootKeyShortcutModal';
import {
  hasPressedCommandAndForwardSlash,
  isEscape,
} from 'shared/helpers/KeyboardHelpers';
import eventListenerMixins from 'shared/mixins/eventListenerMixins';

export default {
  components: {
    AgentDetails,
    SidebarItem,
    AvailabilityStatus,
    NotificationBell,
    OptionsMenu,
    AccountSelector,
    AddAccountModal,
    AddLabelModal,
    WootKeyShortcutModal,
  },
  mixins: [adminMixin, alertMixin, eventListenerMixins],
  data() {
    return {
      showOptionsMenu: false,
      showAccountModal: false,
      showCreateAccountModal: false,
      showAddLabelModal: false,
      showShortcutModal: false,
    };
  },

  computed: {
    ...mapGetters({
      currentUser: 'getCurrentUser',
      globalConfig: 'globalConfig/get',
      inboxes: 'inboxes/getInboxes',
      accountId: 'getCurrentAccountId',
      currentRole: 'getCurrentRole',
      accountLabels: 'labels/getLabelsOnSidebar',
      teams: 'teams/getMyTeams',
    }),

    sidemenuItems() {
      return getSidebarItems(this.accountId);
    },
    accessibleMenuItems() {
      // get all keys in menuGroup
      const groupKey = Object.keys(this.sidemenuItems);

      let menuItems = [];
      // Iterate over menuGroup to find the correct group
      for (let i = 0; i < groupKey.length; i += 1) {
        const groupItem = this.sidemenuItems[groupKey[i]];
        // Check if current route is included
        const isRouteIncluded = groupItem.routes.includes(this.currentRoute);
        if (isRouteIncluded) {
          menuItems = Object.values(groupItem.menuItems);
        }
      }

      return this.filterMenuItemsByRole(menuItems);
    },
    currentRoute() {
      return this.$store.state.route.name;
    },
    shouldShowSidebarItem() {
      return this.sidemenuItems.common.routes.includes(this.currentRoute);
    },
    showShowContactSideMenu() {
      return this.sidemenuItems.contacts.routes.includes(this.currentRoute);
    },
    shouldShowTeams() {
      return this.shouldShowSidebarItem && this.teams.length;
    },
    inboxSection() {
      return {
        icon: 'ion-folder',
        label: 'INBOXES',
        hasSubMenu: true,
        newLink: true,
        key: 'inbox',
        cssClass: 'menu-title align-justify',
        toState: frontendURL(`accounts/${this.accountId}/settings/inboxes`),
        toStateName: 'settings_inbox_list',
        newLinkRouteName: 'settings_inbox_new',
        children: this.inboxes.map(inbox => ({
          id: inbox.id,
          label: inbox.name,
          toState: frontendURL(`accounts/${this.accountId}/inbox/${inbox.id}`),
          type: inbox.channel_type,
          phoneNumber: inbox.phone_number,
        })),
      };
    },
    labelSection() {
      return {
        icon: 'ion-pound',
        label: 'LABELS',
        hasSubMenu: true,
        newLink: true,
        key: 'label',
        cssClass: 'menu-title align-justify',
        toState: frontendURL(`accounts/${this.accountId}/settings/labels`),
        toStateName: 'labels_list',
        showModalForNewItem: true,
        modalName: 'AddLabel',
        children: this.accountLabels.map(label => ({
          id: label.id,
          label: label.title,
          color: label.color,
          truncateLabel: true,
          toState: frontendURL(
            `accounts/${this.accountId}/label/${label.title}`
          ),
        })),
      };
    },
    contactLabelSection() {
      return {
        icon: 'ion-pound',
        label: 'TAGGED_WITH',
        hasSubMenu: true,
        key: 'label',
        newLink: false,
        cssClass: 'menu-title align-justify',
        toState: frontendURL(`accounts/${this.accountId}/settings/labels`),
        toStateName: 'labels_list',
        showModalForNewItem: true,
        modalName: 'AddLabel',
        children: this.accountLabels.map(label => ({
          id: label.id,
          label: label.title,
          color: label.color,
          truncateLabel: true,
          toState: frontendURL(
            `accounts/${this.accountId}/labels/${label.title}/contacts`
          ),
        })),
      };
    },
    teamSection() {
      return {
        icon: 'ion-ios-people',
        label: 'TEAMS',
        hasSubMenu: true,
        newLink: true,
        key: 'team',
        cssClass: 'menu-title align-justify teams-sidebar-menu',
        toState: frontendURL(`accounts/${this.accountId}/settings/teams`),
        toStateName: 'teams_list',
        newLinkRouteName: 'settings_teams_new',
        children: this.teams.map(team => ({
          id: team.id,
          label: team.name,
          truncateLabel: true,
          toState: frontendURL(`accounts/${this.accountId}/team/${team.id}`),
        })),
      };
    },
    dashboardPath() {
      return frontendURL(`accounts/${this.accountId}/dashboard`);
    },
  },
  watch: {
    currentUser(newUserInfo, oldUserInfo) {
      if (!oldUserInfo.email && newUserInfo.email) {
        this.setChatwootUser();
      }
    },
  },
  mounted() {
    this.$store.dispatch('labels/get');
    this.$store.dispatch('inboxes/get');
    this.$store.dispatch('notifications/unReadCount');
    this.$store.dispatch('teams/get');
    this.setChatwootUser();
  },

  methods: {
    toggleKeyShortcutModal() {
      this.showShortcutModal = true;
    },
    closeKeyShortcutModal() {
      this.showShortcutModal = false;
    },
    handleKeyEvents(e) {
      if (hasPressedCommandAndForwardSlash(e)) {
        this.toggleKeyShortcutModal();
      }
      if (isEscape(e)) {
        this.closeKeyShortcutModal();
      }
    },
    toggleSupportChatWindow() {
      window.$chatwoot.toggle();
    },
    setChatwootUser() {
      if (!this.currentUser.email || !this.globalConfig.chatwootInboxToken) {
        return;
      }
      window.$chatwoot.setUser(this.currentUser.email, {
        name: this.currentUser.name,
        email: this.currentUser.email,
        avatar_url: this.currentUser.avatar_url,
        identifier_hash: this.currentUser.hmac_identifier,
      });
    },
    filterMenuItemsByRole(menuItems) {
      if (!this.currentRole) {
        return [];
      }
      return menuItems.filter(
        menuItem =>
          window.roleWiseRoutes[this.currentRole].indexOf(
            menuItem.toStateName
          ) > -1
      );
    },
    toggleOptions() {
      this.showOptionsMenu = !this.showOptionsMenu;
    },
    toggleAccountModal() {
      this.showAccountModal = !this.showAccountModal;
    },
    openCreateAccountModal() {
      this.showAccountModal = false;
      this.showCreateAccountModal = true;
    },
    closeCreateAccountModal() {
      this.showCreateAccountModal = false;
    },
    showAddLabelPopup() {
      this.showAddLabelModal = true;
    },
    hideAddLabelPopup() {
      this.showAddLabelModal = false;
    },
  },
};
</script>

<style lang="scss">
@import '~dashboard/assets/scss/variables';

.account-selector--modal {
  .modal-container {
    width: 40rem;
  }
}

.account-selector {
  cursor: pointer;
  padding: $space-small $space-large;

  .ion-ios-checkmark {
    font-size: $font-size-big;

    & + .account--details {
      padding-left: $space-normal;
    }
  }

  .account--details {
    padding-left: $space-large + $space-smaller;
  }

  &:last-child {
    margin-bottom: $space-large;
  }

  a {
    align-items: center;
    cursor: pointer;
    display: flex;

    .account--name {
      cursor: pointer;
      font-size: $font-size-medium;
      font-weight: $font-weight-medium;
      line-height: 1;
    }

    .account--role {
      cursor: pointer;
      font-size: $font-size-mini;
      text-transform: capitalize;
    }
  }
}

.app-context-menu {
  align-items: center;
  cursor: pointer;
  display: flex;
  flex-direction: row;
  height: 6rem;
}

.current-user--options {
  font-size: $font-size-big;
  margin-bottom: auto;
  margin-left: auto;
  margin-top: auto;
}

.teams-sidebar-menu + .nested.vertical.menu {
  padding-left: calc(var(--space-medium) - var(--space-one));
}
</style>
