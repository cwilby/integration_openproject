<template>
	<div class="openproject-prefs section">
		<SettingsTitle />
		<div v-if="connected" class="openproject-prefs--connected">
			<label>
				<CheckIcon :size="20" />
				{{ t('integration_openproject', 'Connected as {user}', { user: state.user_name }) }}
			</label>
			<NcButton class="openproject-prefs--disconnect" @click="disconnectFromOP()">
				<template #icon>
					<CloseIcon :size="23" />
				</template>
				{{ t('integration_openproject', 'Disconnect from OpenProject') }}
			</NcButton>
		</div>
		<br>
		<div v-if="connected" class="openproject-prefs--form">
			<CheckBox v-model="state.navigation_enabled"
				input-id="openproject-prefs--link"
				:label="t('integration_openproject', 'Enable navigation link')" />
			<CheckBox v-model="state.search_enabled"
				input-id="openproject-prefs--u-search"
				:label="t('integration_openproject', 'Enable unified search for tickets')">
				<template #hint>
					<p v-if="state.search_enabled" class="openproject-prefs--hint">
						<InformationVariant />
						{{ t('integration_openproject', 'Warning, everything you type in the search bar will be sent to your OpenProject instance.') }}
					</p>
					<br v-else>
				</template>
			</CheckBox>
		</div>
		<OAuthConnectButton v-else :is-admin-config-ok="state.admin_config_ok" />
	</div>
</template>

<script>
import { loadState } from '@nextcloud/initial-state'
import { generateUrl } from '@nextcloud/router'
import axios from '@nextcloud/axios'
import CloseIcon from 'vue-material-design-icons/Close.vue'
import CheckIcon from 'vue-material-design-icons/Check.vue'
import InformationVariant from 'vue-material-design-icons/InformationVariant.vue'
import { showSuccess, showError } from '@nextcloud/dialogs'
import SettingsTitle from '../components/settings/SettingsTitle.vue'
import OAuthConnectButton from './OAuthConnectButton.vue'
import CheckBox from './settings/CheckBox.vue'
import { translate as t } from '@nextcloud/l10n'
import { checkOauthConnectionResult } from '../utils.js'
import NcButton from '@nextcloud/vue/dist/Components/NcButton.js'

export default {
	name: 'PersonalSettings',

	components: {
		SettingsTitle, OAuthConnectButton, NcButton, CloseIcon, CheckIcon, InformationVariant, CheckBox,
	},

	data() {
		return {
			loading: false,
			state: loadState('integration_openproject', 'user-config'),
			oauthConnectionErrorMessage: loadState('integration_openproject', 'oauth-connection-error-message'),
			oauthConnectionResult: loadState('integration_openproject', 'oauth-connection-result'),
		}
	},
	computed: {
		connected() {
			if (!this.state.admin_config_ok) return false
			return this.state.token && this.state.token !== ''
				&& this.state.user_name && this.state.user_name !== ''
		},
	},
	watch: {
		'state.search_enabled'(newVal) {
			this.saveOptions({
				search_enabled: newVal ? '1' : '0',
			})
		},
		'state.navigation_enabled'(newVal) {
			this.saveOptions({
				navigation_enabled: newVal ? '1' : '0',
			})
		},
	},

	mounted() {
		checkOauthConnectionResult(this.oauthConnectionResult, this.oauthConnectionErrorMessage)
	},

	methods: {
		disconnectFromOP() {
			this.state.token = ''
			this.saveOptions({ token: this.state.token, token_type: '' })
		},
		saveOptions(values) {
			const req = {
				values,
			}
			const url = generateUrl('/apps/integration_openproject/config')
			axios.put(url, req)
				.then((response) => {
					showSuccess(t('integration_openproject', 'OpenProject options saved'))
					if (response.data.user_name !== undefined) {
						this.state.user_name = response.data.user_name
						if (this.state.token && response.data.user_name === '') {
							showError(t('integration_openproject', 'Incorrect access token'))
						}
					}
				})
				.catch((error) => {
					console.debug(error.response)
					const msg = error.response?.data?.errorMessage === 'Invalid token'
						? t('integration_openproject', 'Invalid token')
						: error.response?.data?.errorMessage === 'Not found'
							? t('integration_openproject', 'OpenProject instance not found')
							: error.response?.request?.responseText
					showError(
						t('integration_openproject', 'Failed to save OpenProject options')
						+ ': ' + msg
					)
				})
				.finally(() => {
					this.loading = false
				})
		},
	},
}
</script>

<style scoped lang="scss">
.openproject-prefs {
	&--connected {
		padding-block: 1rem;
		label {
			display: flex;
			align-items: center;
			padding-bottom: .5rem;
		}
		.check-icon {
			padding-right: .2rem;
			color: var(--color-success);
		}
	}
	&--hint {
		display: flex;
		align-items: center;
		padding-top: 1rem;
	}
	.oauth-connect--message {
		text-align: left;
		padding: 0;
	}
}
</style>
