<template>
	<AuthView
		v-if="config"
		:form="config"
		:formLoading="loading"
		@submit="onSubmit"
		@input="onInput"
	/>
</template>

<script lang="ts">
import AuthView from '@/views/AuthView.vue';
import { useToast } from '@/composables';

import { defineComponent } from 'vue';
import type { IFormBoxConfig } from '@/Interface';
import { VIEWS } from '@/constants';
import { mapStores } from 'pinia';
import { useUsersStore } from '@/stores/users.store';

export default defineComponent({
	name: 'ChangePasswordView',
	components: {
		AuthView,
	},
	setup() {
		return {
			...useToast(),
		};
	},
	data() {
		return {
			password: '',
			loading: false,
			config: null as null | IFormBoxConfig,
		};
	},
	computed: {
		...mapStores(useUsersStore),
	},
	async mounted() {
		this.config = {
			title: this.$locale.baseText('auth.changePassword'),
			buttonText: this.$locale.baseText('auth.changePassword'),
			redirectText: this.$locale.baseText('auth.signin'),
			redirectLink: '/signin',
			inputs: [
				{
					name: 'password',
					properties: {
						label: this.$locale.baseText('auth.newPassword'),
						type: 'password',
						required: true,
						validationRules: [{ name: 'DEFAULT_PASSWORD_RULES' }],
						infoText: this.$locale.baseText('auth.defaultPasswordRequirements'),
						autocomplete: 'new-password',
						capitalize: true,
					},
				},
				{
					name: 'password2',
					properties: {
						label: this.$locale.baseText('auth.changePassword.reenterNewPassword'),
						type: 'password',
						required: true,
						validators: {
							TWO_PASSWORDS_MATCH: {
								validate: this.passwordsMatch,
							},
						},
						validationRules: [{ name: 'TWO_PASSWORDS_MATCH' }],
						autocomplete: 'new-password',
						capitalize: true,
					},
				},
			],
		};

		const token = this.getResetToken();

		try {
			if (!token) {
				throw new Error(this.$locale.baseText('auth.changePassword.missingTokenError'));
			}

			await this.usersStore.validatePasswordToken({ token });
		} catch (e) {
			this.showMessage({
				title: this.$locale.baseText('auth.changePassword.tokenValidationError'),
				type: 'error',
			});
		}
	},
	methods: {
		passwordsMatch(value: string | number | boolean | null | undefined) {
			if (typeof value !== 'string') {
				return false;
			}

			if (value !== this.password) {
				return {
					messageKey: 'auth.changePassword.passwordsMustMatchError',
				};
			}

			return false;
		},
		onInput(e: { name: string; value: string }) {
			if (e.name === 'password') {
				this.password = e.value;
			}
		},
		getResetToken(): string | null {
			return !this.$route.query.token || typeof this.$route.query.token !== 'string'
				? null
				: this.$route.query.token;
		},
		async onSubmit() {
			try {
				this.loading = true;
				const token = this.getResetToken();

				if (token) {
					await this.usersStore.changePassword({ token, password: this.password });

					this.showMessage({
						type: 'success',
						title: this.$locale.baseText('auth.changePassword.passwordUpdated'),
						message: this.$locale.baseText('auth.changePassword.passwordUpdatedMessage'),
					});

					await this.$router.push({ name: VIEWS.SIGNIN });
				} else {
					this.showError(
						new Error(this.$locale.baseText('auth.validation.missingParameters')),
						this.$locale.baseText('auth.changePassword.error'),
					);
				}
			} catch (error) {
				this.showError(error, this.$locale.baseText('auth.changePassword.error'));
			}
			this.loading = false;
		},
	},
});
</script>
