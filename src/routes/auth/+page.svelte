<script lang="ts">
	import DOMPurify from 'dompurify';
	import { marked } from 'marked';

	import { toast } from 'svelte-sonner';

	import { onMount, getContext } from 'svelte';
	import { goto } from '$app/navigation';
	import { page } from '$app/stores';
	import { env } from '$env/dynamic/public';

	import { WEBUI_API_BASE_URL, WEBUI_BASE_URL } from '$lib/constants';
	import { WEBUI_NAME, config, user } from '$lib/stores';

	import Spinner from '$lib/components/common/Spinner.svelte';
	import OnBoarding from '$lib/components/OnBoarding.svelte';
	import SensitiveInput from '$lib/components/common/SensitiveInput.svelte';

	const i18n = getContext('i18n');

	let loaded = false;

	let mode = $config?.features.enable_ldap ? 'ldap' : 'signin';

	let form = null;

	let name = '';
	let email = '';
	let password = '';
	let confirmPassword = '';

	let ldapUsername = '';

	let showPassword = false;
	let errorMsg = '';
	const AUTH_TEMP_API_BASE_URL = env.PUBLIC_AUTH_TEMP_API_BASE_URL || WEBUI_API_BASE_URL;
	const videoExtensions = /\.(mp4|webm|ogg|mov)(\?.*)?$/i;
	const DEFAULT_BG_COLOR = '#0f131c';
	const AUTH_TEMP_BG_MEDIA = env.PUBLIC_AUTH_TEMP_BG_MEDIA || '/background/landing.mp4';

	let backgroundMedia = '';
	let headerBgColor = '#f2f4f8';
	let headerToneClass = 'is-dark';
	let wrapperStyle = '';
	let isVideoBackground = false;

	const authTempRequest = async (path: string, payload: Record<string, unknown>) => {
		const res = await fetch(`${AUTH_TEMP_API_BASE_URL}${path}`, {
			method: 'POST',
			headers: {
				Accept: 'application/json',
				'Content-Type': 'application/json'
			},
			body: JSON.stringify(payload)
		});

		if (!res.ok) {
			let message = 'Request failed';
			try {
				const data = await res.json();
				message = data?.detail || data?.error || message;
			} catch {
				// no-op
			}
			throw new Error(message);
		}

		return await res.json();
	};


	const submitHandler = async () => {
		errorMsg = '';

		try {
			let sessionUser = null;
			if (mode === 'ldap') {
				sessionUser = await authTempRequest('/auths/signin', {
					ldap: true,
					username: ldapUsername,
					password
				});
			} else if (mode === 'signup') {
				if ($config?.features?.enable_signup_password_confirmation && password !== confirmPassword) {
					errorMsg = $i18n.t('Passwords do not match.');
					return;
				}
				sessionUser = await authTempRequest('/auths/signup', {
					name,
					email,
					password
				});
			} else {
				sessionUser = await authTempRequest('/auths/signin', { email, password });
			}

			if (sessionUser?.token) {
				localStorage.token = sessionUser.token;
			}
			await user.set(sessionUser);
			toast.success($i18n.t(`You're now logged in.`));
			goto($page.url.searchParams.get('redirect') || '/');
		} catch (error) {
			errorMsg = `${error}`;
		}
	};

	let onboarding = false;

	const getHeaderToneClass = (hexColor: string) => {
		const hex = hexColor.replace('#', '');
		const normalized = hex.length === 3 ? hex.split('').map((c) => c + c).join('') : hex;
		if (!/^[0-9a-fA-F]{6}$/.test(normalized)) {
			return 'is-dark';
		}
		const r = parseInt(normalized.slice(0, 2), 16);
		const g = parseInt(normalized.slice(2, 4), 16);
		const b = parseInt(normalized.slice(4, 6), 16);
		const luminance = (0.299 * r + 0.587 * g + 0.114 * b) / 255;
		return luminance < 0.6 ? 'is-dark' : 'is-light';
	};

	onMount(async () => {
		const error = $page.url.searchParams.get('error');
		if (error) {
			toast.error(error);
		}
		form = $page.url.searchParams.get('form');

		loaded = true;
		onboarding = $config?.onboarding ?? false;
	});

	$: backgroundMedia = AUTH_TEMP_BG_MEDIA;
	$: headerBgColor = $config?.metadata?.auth_header_bg ?? '#f2f4f8';
	$: headerToneClass = getHeaderToneClass(headerBgColor);
	$: isVideoBackground = !!backgroundMedia && videoExtensions.test(backgroundMedia);
	$: wrapperStyle = isVideoBackground
		? `background-color: ${DEFAULT_BG_COLOR};`
		: backgroundMedia
			? `background-color: ${DEFAULT_BG_COLOR}; background-image: url('${backgroundMedia}'); background-size: cover; background-position: center;`
			: `background-color: ${DEFAULT_BG_COLOR};`;
</script>

<svelte:head>
	<title>
		{`${$WEBUI_NAME}`}
	</title>
</svelte:head>

<OnBoarding
	bind:show={onboarding}
	getStartedHandler={() => {
		onboarding = false;
		mode = $config?.features.enable_ldap ? 'ldap' : 'signup';
	}}
/>

<div class="auth-temp-wrapper" style={wrapperStyle}>
	{#if isVideoBackground}
		<video src={backgroundMedia} autoplay loop muted playsinline class="bg-video"></video>
	{/if}
	<div class="bg-blob bg-blob--topleft"></div>
	<div class="bg-blob bg-blob--bottomright"></div>

	<div class="drag-region" />

	{#if loaded}
		<div class="auth-temp-container">
			<div class="auth-temp-card">
				{#if ($config?.features.auth_trusted_header ?? false) || $config?.features.auth === false}
					<div class="flex items-center justify-center gap-3 text-xl text-center font-medium">
						<span>{$i18n.t('Signing in to {{WEBUI_NAME}}', { WEBUI_NAME: $WEBUI_NAME })}</span>
						<Spinner className="size-5" />
					</div>
				{:else}
					<div class={`card-header ${headerToneClass}`}>
						<h1 class="card-title">
							{#if $config?.onboarding ?? false}
								{$i18n.t(`Get started with {{WEBUI_NAME}}`, { WEBUI_NAME: $WEBUI_NAME })}
							{:else if mode === 'ldap'}
								{$i18n.t(`Sign in to {{WEBUI_NAME}} with LDAP`, { WEBUI_NAME: $WEBUI_NAME })}
							{:else if mode === 'signin'}
								{$i18n.t(`Sign in to {{WEBUI_NAME}}`, { WEBUI_NAME: $WEBUI_NAME })}
							{:else}
								{$i18n.t(`Sign up to {{WEBUI_NAME}}`, { WEBUI_NAME: $WEBUI_NAME })}
							{/if}
						</h1>

						{#if $config?.onboarding ?? false}
							<p class="onboarding-info">
								{$WEBUI_NAME}
								{$i18n.t(
									'does not make any external connections, and your data stays securely on your locally hosted server.'
								)}
							</p>
						{/if}
					</div>

					<form
						class="auth-form"
						on:submit={(e) => {
							e.preventDefault();
							submitHandler();
						}}
					>
							{#if mode === 'signup'}
								<div class="form-field">
									<label for="name" class="form-label">{$i18n.t('Name')}</label>
									<input
										bind:value={name}
										type="text"
										id="name"
										class="glass-input"
										autocomplete="name"
										placeholder={$i18n.t('Enter Your Full Name')}
										required
									/>
								</div>
							{/if}

							{#if mode === 'ldap'}
								<div class="form-field">
									<label for="username" class="form-label">{$i18n.t('Username')}</label>
									<input
										bind:value={ldapUsername}
										type="text"
										id="username"
										class="glass-input"
										autocomplete="username"
										name="username"
										placeholder={$i18n.t('Enter Your Username')}
										required
									/>
								</div>
							{:else}
								<div class="form-field">
									<label for="email" class="form-label">{$i18n.t('Email')}</label>
									<input
										bind:value={email}
										type="email"
										id="email"
										class="glass-input {mode === 'signup' ? 'signup-email-input' : ''}"
										autocomplete="email"
										name="email"
										maxlength="28"
										placeholder={$i18n.t('Enter Your Email')}
										required
									/>
								</div>
							{/if}

							<div class="form-field">
								<label for="password" class="form-label">{$i18n.t('Password')}</label>
								<div class="password-input-wrapper">
									<input
										bind:value={password}
										type={showPassword ? 'text' : 'password'}
										id="password"
										class="glass-input password-input"
										placeholder={$i18n.t('Enter Your Password')}
										autocomplete={mode === 'signup' ? 'new-password' : 'current-password'}
										name="password"
										maxlength="28"
										required
									/>
									<button
										type="button"
										class="password-toggle-btn"
										aria-label={showPassword
											? $i18n.t('Hide password')
											: $i18n.t('Show password')}
										on:mousedown={() => (showPassword = true)}
										on:mouseup={() => (showPassword = false)}
										on:mouseleave={() => (showPassword = false)}
										on:touchstart={() => (showPassword = true)}
										on:touchend={() => (showPassword = false)}
										on:keydown={(e) => {
											if (e.key === 'Enter' || e.key === ' ') {
												e.preventDefault();
												showPassword = !showPassword;
											}
										}}
									>
										{#if showPassword}
											<svg
												xmlns="http://www.w3.org/2000/svg"
												viewBox="0 0 16 16"
												fill="currentColor"
												class="size-4"
												aria-hidden="true"
											>
												<path
													fill-rule="evenodd"
													d="M3.28 2.22a.75.75 0 0 0-1.06 1.06l10.5 10.5a.75.75 0 1 0 1.06-1.06l-1.322-1.323a7.012 7.012 0 0 0 2.16-3.11.87.87 0 0 0 0-.567A7.003 7.003 0 0 0 4.82 3.76l-1.54-1.54Zm3.196 3.195 1.135 1.136A1.502 1.502 0 0 1 9.45 8.389l1.136 1.135a3 3 0 0 0-4.109-4.109Z"
													clip-rule="evenodd"
												/>
												<path
													d="m7.812 10.994 1.816 1.816A7.003 7.003 0 0 1 1.38 8.28a.87.87 0 0 1 0-.566 6.985 6.985 0 0 1 1.113-2.039l2.513 2.513a3 3 0 0 0 2.806 2.806Z"
												/>
											</svg>
										{:else}
											<svg
												xmlns="http://www.w3.org/2000/svg"
												viewBox="0 0 16 16"
												fill="currentColor"
												class="size-4"
												aria-hidden="true"
											>
												<path d="M8 9.5a1.5 1.5 0 1 0 0-3 1.5 1.5 0 0 0 0 3Z" />
												<path
													fill-rule="evenodd"
													d="M1.38 8.28a.87.87 0 0 1 0-.566 7.003 7.003 0 0 1 13.238.006.87.87 0 0 1 0 .566A7.003 7.003 0 0 1 1.379 8.28ZM11 8a3 3 0 1 1-6 0 3 3 0 0 1 6 0Z"
													clip-rule="evenodd"
												/>
											</svg>
										{/if}
									</button>
								</div>
							</div>

							{#if mode === 'signup' && $config?.features?.enable_signup_password_confirmation}
								<div class="form-field">
									<label for="confirm-password" class="form-label"
										>{$i18n.t('Confirm Password')}</label
									>
									<SensitiveInput
										bind:value={confirmPassword}
										type="password"
										id="confirm-password"
										outerClassName="flex flex-1 bg-transparent"
										inputClassName="w-full text-sm py-0.5 bg-transparent"
										placeholder={$i18n.t('Confirm Your Password')}
										autocomplete="new-password"
										name="confirm-password"
										maxlength="28"
										screenReader={false}
										required
									/>
								</div>
							{/if}

							{#if errorMsg}
								<div class="error-message">{errorMsg}</div>
							{/if}

							<button type="submit" class="submit-btn">
								{mode === 'ldap'
									? $i18n.t('Authenticate')
									: mode === 'signin'
										? $i18n.t('Sign in')
										: ($config?.onboarding ?? false)
											? $i18n.t('Create Admin Account')
											: $i18n.t('Create Account')}
							</button>

							{#if ($config?.features?.enable_signup ?? true) && !($config?.onboarding ?? false) && mode !== 'ldap'}
								<div class="mode-toggle">
									{mode === 'signin'
										? $i18n.t("Don't have an account?")
										: $i18n.t('Already have an account?')}

									<button
										class="mode-toggle-btn"
										type="button"
										on:click={() => {
											if (mode === 'signin') {
												mode = 'signup';
											} else {
												mode = 'signin';
											}
										}}
									>
										{mode === 'signin' ? $i18n.t('Sign up') : $i18n.t('Sign in')}
									</button>
								</div>
							{/if}
						</form>

					{#if Object.keys($config?.oauth?.providers ?? {}).length > 0}
						<div class="oauth-divider">
							<hr class="oauth-divider-line" />
							{#if $config?.features.enable_login_form || $config?.features.enable_ldap || form}
								<span class="oauth-divider-text">{$i18n.t('or')}</span>
							{/if}
							<hr class="oauth-divider-line" />
						</div>
						<div class="oauth-buttons">
							{#if $config?.oauth?.providers?.google}
								<button
									class="oauth-btn"
									on:click={() => {
										window.location.href = `${WEBUI_BASE_URL}/oauth/google/login`;
									}}
								>
									<svg
										xmlns="http://www.w3.org/2000/svg"
										viewBox="0 0 48 48"
										class="size-5"
										aria-hidden="true"
									>
										<path
											fill="#EA4335"
											d="M24 9.5c3.54 0 6.71 1.22 9.21 3.6l6.85-6.85C35.9 2.38 30.47 0 24 0 14.62 0 6.51 5.38 2.56 13.22l7.98 6.19C12.43 13.72 17.74 9.5 24 9.5z"
										/><path
											fill="#4285F4"
											d="M46.98 24.55c0-1.57-.15-3.09-.38-4.55H24v9.02h12.94c-.58 2.96-2.26 5.48-4.78 7.18l7.73 6c4.51-4.18 7.09-10.36 7.09-17.65z"
										/><path
											fill="#FBBC05"
											d="M10.53 28.59c-.48-1.45-.76-2.99-.76-4.59s.27-3.14.76-4.59l-7.98-6.19C.92 16.46 0 20.12 0 24c0 3.88.92 7.54 2.56 10.78l7.97-6.19z"
										/><path
											fill="#34A853"
											d="M24 48c6.48 0 11.93-2.13 15.89-5.81l-7.73-6c-2.15 1.45-4.92 2.3-8.16 2.3-6.26 0-11.57-4.22-13.47-9.91l-7.98 6.19C6.51 42.62 14.62 48 24 48z"
										/><path fill="none" d="M0 0h48v48H0z" />
									</svg>
									<span>{$i18n.t('Continue with {{provider}}', { provider: 'Google' })}</span>
								</button>
							{/if}
							{#if $config?.oauth?.providers?.microsoft}
								<button
									class="oauth-btn"
									on:click={() => {
										window.location.href = `${WEBUI_BASE_URL}/oauth/microsoft/login`;
									}}
								>
									<svg
										xmlns="http://www.w3.org/2000/svg"
										viewBox="0 0 21 21"
										class="size-5"
										aria-hidden="true"
									>
										<rect x="1" y="1" width="9" height="9" fill="#f25022" /><rect
											x="1"
											y="11"
											width="9"
											height="9"
											fill="#00a4ef"
										/><rect x="11" y="1" width="9" height="9" fill="#7fba00" /><rect
											x="11"
											y="11"
											width="9"
											height="9"
											fill="#ffb900"
										/>
									</svg>
									<span>{$i18n.t('Continue with {{provider}}', { provider: 'Microsoft' })}</span
									>
								</button>
							{/if}
							{#if $config?.oauth?.providers?.github}
								<button
									class="oauth-btn"
									on:click={() => {
										window.location.href = `${WEBUI_BASE_URL}/oauth/github/login`;
									}}
								>
									<svg
										xmlns="http://www.w3.org/2000/svg"
										viewBox="0 0 24 24"
										class="size-5"
										aria-hidden="true"
									>
										<path
											fill="currentColor"
											d="M12 0C5.37 0 0 5.37 0 12c0 5.31 3.435 9.795 8.205 11.385.6.105.825-.255.825-.57 0-.285-.015-1.23-.015-2.235-3.015.555-3.795-.735-4.035-1.41-.135-.345-.72-1.41-1.23-1.695-.42-.225-1.02-.78-.015-.795.945-.015 1.62.87 1.845 1.23 1.08 1.815 2.805 1.305 3.495.99.105-.78.42-1.305.765-1.605-2.67-.3-5.46-1.335-5.46-5.925 0-1.305.465-2.385 1.23-3.225-.12-.3-.54-1.53.12-3.18 0 0 1.005-.315 3.3 1.23.96-.27 1.98-.405 3-.405s2.04.135 3 .405c2.295-1.56 3.3-1.23 3.3-1.23.66 1.65.24 2.88.12 3.18.765.84 1.23 1.92 1.23 3.225 0 4.605-2.805 5.625-5.475 5.925.435.375.81 1.095.81 2.22 0 1.605-.015 2.895-.015 3.3 0 .315.225.69.825.57C20.565 21.795 24 17.31 24 12c0-6.63-5.37-12-12-12z"
										/>
									</svg>
									<span>{$i18n.t('Continue with {{provider}}', { provider: 'GitHub' })}</span>
								</button>
							{/if}
							{#if $config?.oauth?.providers?.oidc}
								<button
									class="oauth-btn"
									on:click={() => {
										window.location.href = `${WEBUI_BASE_URL}/oauth/oidc/login`;
									}}
								>
									<svg
										xmlns="http://www.w3.org/2000/svg"
										fill="none"
										viewBox="0 0 24 24"
										stroke-width="1.5"
										stroke="currentColor"
										class="size-5"
										aria-hidden="true"
									>
										<path
											stroke-linecap="round"
											stroke-linejoin="round"
											d="M15.75 5.25a3 3 0 0 1 3 3m3 0a6 6 0 0 1-7.029 5.912c-.563-.097-1.159.026-1.563.43L10.5 17.25H8.25v2.25H6v2.25H2.25v-2.818c0-.597.237-1.17.659-1.591l6.499-6.499c.404-.404.527-1 .43-1.563A6 6 0 1 1 21.75 8.25Z"
										/>
									</svg>

									<span
										>{$i18n.t('Continue with {{provider}}', {
											provider: $config?.oauth?.providers?.oidc ?? 'SSO'
										})}</span
									>
								</button>
							{/if}
							{#if $config?.oauth?.providers?.feishu}
								<button
									class="oauth-btn"
									on:click={() => {
										window.location.href = `${WEBUI_BASE_URL}/oauth/feishu/login`;
									}}
								>
									<span>{$i18n.t('Continue with {{provider}}', { provider: 'Feishu' })}</span>
								</button>
							{/if}
						</div>
					{/if}

					{#if $config?.features.enable_ldap && $config?.features.enable_login_form}
						<div class="ldap-toggle-wrapper">
							<button
								class="ldap-toggle-btn"
								type="button"
								on:click={() => {
									if (mode === 'ldap')
										mode = ($config?.onboarding ?? false) ? 'signup' : 'signin';
									else mode = 'ldap';
								}}
							>
								<span
									>{mode === 'ldap'
										? $i18n.t('Continue with Email')
										: $i18n.t('Continue with LDAP')}</span
								>
							</button>
						</div>
					{/if}
				{/if}
			</div>

			{#if $config?.metadata?.login_footer}
				<div class="login-footer">
					<div class="login-footer-content marked">
						{@html DOMPurify.sanitize(marked($config?.metadata?.login_footer))}
					</div>
				</div>
			{/if}
		</div>
	{/if}
</div>

<style>
	.auth-temp-wrapper {
		position: relative;
		min-height: 100vh;
		min-height: 100dvh;
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 24px;
		overflow: hidden;
		transition: background 0.3s ease;
		background-image:
			radial-gradient(circle at 8% 10%, rgba(103, 80, 164, 0.14) 0, rgba(103, 80, 164, 0) 40%),
			radial-gradient(circle at 92% 90%, rgba(56, 142, 60, 0.1) 0, rgba(56, 142, 60, 0) 38%);
	}

	.bg-video {
		position: absolute;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		object-fit: cover;
		z-index: 0;
		pointer-events: none;
	}

	/* Animated background blobs */
	.bg-blob {
		position: absolute;
		width: 600px;
		height: 600px;
		border-radius: 50%;
		filter: blur(80px);
		opacity: 0.28;
		pointer-events: none;
		animation: blob-drift 20s ease-in-out infinite alternate;
	}

	.bg-blob--topleft {
		top: -200px;
		left: -200px;
		background: radial-gradient(circle, rgba(103, 80, 164, 0.6), transparent 70%);
		animation-delay: 0s;
	}

	.bg-blob--bottomright {
		bottom: -200px;
		right: -200px;
		background: radial-gradient(circle, rgba(56, 142, 60, 0.4), transparent 70%);
		animation-delay: -10s;
	}

	@keyframes blob-drift {
		0% {
			transform: translate(0, 0) scale(1);
		}
		33% {
			transform: translate(30px, -30px) scale(1.1);
		}
		66% {
			transform: translate(-20px, 20px) scale(0.9);
		}
		100% {
			transform: translate(20px, -10px) scale(1.05);
		}
	}

	.drag-region {
		position: absolute;
		top: 0;
		left: 0;
		right: 0;
		height: 32px;
		z-index: 50;
		-webkit-app-region: drag;
	}

	.auth-temp-container {
		position: relative;
		z-index: 1;
		width: 100%;
		max-width: 440px;
		display: flex;
		flex-direction: column;
		align-items: center;
	}

	.auth-temp-card {
		--md-sys-color-primary: #6750a4;
		--md-sys-color-surface-container: #f3edf7;
		--md-sys-color-on-surface: #1d1b20;
		--md-sys-color-on-surface-variant: #49454f;
		--md-sys-color-error: #ba1a1a;
		width: 100%;
		background:
			linear-gradient(
			165deg,
			rgba(255, 255, 255, 0.12) 0%,
			rgba(255, 255, 255, 0.07) 38%,
			rgba(255, 255, 255, 0.04) 100%
		);
		border-radius: 28px;
		padding: 36px 28px 28px;
		backdrop-filter: blur(6px) saturate(112%);
		-webkit-backdrop-filter: blur(6px) saturate(112%);
		/* border: 1px solid rgba(255, 255, 255, 0.12); */
	}

	.card-header {
		text-align: center;
		margin-bottom: 10px;
		padding: 0;
		border-radius: 16px;
	}

	.card-header.is-dark {
		--header-text-main: #f5f8ff;
	}

	.card-header.is-light {
		--header-text-main: #1b2330;
	}


	.card-title {
		font-size: 1.65rem;
		line-height: 1.2;
		font-weight: 700;
		color: var(--header-text-main, #f5f8ff);
		text-shadow: 0 1px 8px rgba(0, 0, 0, 0.24);
		letter-spacing: 0.01em;
		margin: 0;
	}

	.onboarding-info {
		margin-top: 8px;
		font-size: 0.75rem;
		color: rgba(255, 255, 255, 0.45);
		line-height: 1.4;
	}

	.auth-form {
		display: flex;
		flex-direction: column;
		gap: 2px;
	}

	.form-field {
		margin-bottom: 0;
	}

	.form-label {
		display: none;
	}

	.glass-input {
		min-height: 56px;
		padding: 12px 16px;
		border: 1px solid transparent;
		border-radius: 16px;
		font-size: 15px;
		color: var(--md-sys-color-on-surface);
		background: linear-gradient(
			160deg,
			rgba(245, 250, 255, 0.22) 0%,
			rgba(238, 246, 255, 0.16) 50%,
			rgba(232, 242, 255, 0.12) 100%
		);
		backdrop-filter: blur(3px) saturate(108%);
		-webkit-backdrop-filter: blur(3px) saturate(108%);
		transition: border-color 0.2s, box-shadow 0.2s, background 0.2s;
		outline: none;
		width: 100%;
		box-sizing: border-box;
		position: relative;
		z-index: 1;
	}

	.form-field:first-child .glass-input {
		border-bottom-left-radius: 1.6px;
		border-bottom-right-radius: 1.6px;
	}

	.signup-email-input {
		border-radius: 0;
	}

	.glass-input::placeholder {
		color: color-mix(in srgb, var(--md-sys-color-on-surface-variant) 78%, white 22%);
	}

	.glass-input:focus {
		border-color: transparent;
		box-shadow: 0 0 0 3px color-mix(in srgb, var(--md-sys-color-primary) 24%, transparent);
		background: color-mix(in srgb, var(--md-sys-color-surface-container) 86%, white 14%);
		z-index: 3;
	}

	.password-input-wrapper {
		position: relative;
		display: flex;
		align-items: center;
		margin-top: 0;
	}

	.password-input {
		padding-right: 70px;
		border-radius: 1.6px;
	}

	.password-toggle-btn {
		position: absolute;
		right: 8px;
		top: 50%;
		transform: translateY(-50%);
		z-index: 5;
		background: color-mix(in srgb, var(--md-sys-color-primary) 11%, rgba(255, 255, 255, 0) 89%);
		border: none;
		width: 36px;
		height: 36px;
		border-radius: 50%;
		cursor: pointer;
		display: flex;
		align-items: center;
		justify-content: center;
		color: var(--md-sys-color-primary);
		transition: background 0.2s, color 0.2s;
	}

	.password-toggle-btn:hover {
		background: color-mix(in srgb, var(--md-sys-color-primary) 18%, white 82%);
	}

	/* Remember me checkbox */


	.error-message {
		padding: 12px 14px;
		background: color-mix(in srgb, var(--md-sys-color-error) 9%, white 91%);
		border: 1px solid color-mix(in srgb, var(--md-sys-color-error) 36%, transparent);
		border-radius: 1.6px;
		color: var(--md-sys-color-error);
		font-size: 13px;
		text-align: center;
		margin-top: 2px;
		margin-bottom: 4px;
	}

	.submit-btn {
		min-height: 52px;
		padding: 12px 18px;
		border: 1px solid rgba(255, 255, 255, 0.45);
		border-radius: 14px;
		background: #b89df2;
		color: #5f4b8b;
		font-size: 15px;
		font-weight: 600;
		cursor: pointer;
		transition: transform 0.15s, box-shadow 0.2s, background 0.2s;
		width: 100%;
		margin-top: 0;
		display: flex;
		align-items: center;
		justify-content: center;
		border-top-left-radius: 1.4px;
		border-top-right-radius: 1.4px;
	}

	.submit-btn:hover:not(:disabled) {
		background: #b89df2;
		transform: translateY(-1px);
		filter: brightness(1.03);
	}

	.submit-btn:active:not(:disabled) {
		transform: translateY(0);
	}

	.submit-btn:disabled {
		opacity: 0.5;
		cursor: not-allowed;
	}

	/* Mode toggle (Sign in / Sign up) */
	.mode-toggle {
		margin-top: 16px;
		font-size: 0.8125rem;
		color: rgba(255, 255, 255, 0.5);
		text-align: center;
	}

	.mode-toggle-btn {
		background: none;
		border: none;
		color: rgba(255, 255, 255, 0.8);
		font-size: 0.8125rem;
		font-weight: 500;
		text-decoration: underline;
		cursor: pointer;
		padding: 0;
		margin-left: 4px;
		transition: color 0.2s;
	}

	.mode-toggle-btn:hover {
		color: white;
	}

	/* OAuth section */
	.oauth-divider {
		display: flex;
		align-items: center;
		justify-content: center;
		width: 100%;
		margin: 20px 0 16px;
	}

	.oauth-divider-line {
		flex: 1;
		height: 1px;
		border: none;
		background: rgba(255, 255, 255, 0.08);
	}

	.oauth-divider-text {
		padding: 0 12px;
		font-size: 0.8125rem;
		color: rgba(255, 255, 255, 0.4);
	}

	.oauth-buttons {
		display: flex;
		flex-direction: column;
		gap: 8px;
	}

	.oauth-btn {
		display: flex;
		align-items: center;
		justify-content: center;
		gap: 10px;
		min-height: 46px;
		padding: 10px 16px;
		border: 1px solid rgba(255, 255, 255, 0.1);
		border-radius: 12px;
		background: rgba(255, 255, 255, 0.05);
		color: rgba(255, 255, 255, 0.75);
		font-size: 0.875rem;
		font-weight: 500;
		cursor: pointer;
		transition: background 0.2s, border-color 0.2s, color 0.2s;
		width: 100%;
	}

	.oauth-btn:hover {
		background: rgba(255, 255, 255, 0.1);
		border-color: rgba(255, 255, 255, 0.2);
		color: rgba(255, 255, 255, 0.95);
	}

	/* LDAP toggle */
	.ldap-toggle-wrapper {
		margin-top: 12px;
		text-align: center;
	}

	.ldap-toggle-btn {
		background: none;
		border: none;
		color: rgba(255, 255, 255, 0.45);
		font-size: 0.75rem;
		text-decoration: underline;
		cursor: pointer;
		transition: color 0.2s;
	}

	.ldap-toggle-btn:hover {
		color: rgba(255, 255, 255, 0.7);
	}

	/* Login footer */
	.login-footer {
		margin-top: 20px;
		width: 100%;
		max-width: 440px;
		text-align: center;
	}

	.login-footer-content {
		font-size: 0.7rem;
		color: rgba(255, 255, 255, 0.3);
		line-height: 1.5;
	}

	.login-footer-content :global(a) {
		color: rgba(255, 255, 255, 0.5);
	}

	/* Mobile responsive */
	@media (max-width: 480px) {
		.auth-temp-wrapper {
			padding: 16px;
		}

		.auth-temp-card {
			padding: 28px 18px 24px;
			border-radius: 24px;
		}

		.card-title {
			font-size: 1.125rem;
		}

		.glass-input {
			min-height: 48px;
			padding: 10px 14px;
			font-size: 0.875rem;
			border-radius: 12px;
		}

		.submit-btn {
			min-height: 48px;
			font-size: 0.875rem;
		}

		.oauth-btn {
			min-height: 42px;
			font-size: 0.8125rem;
		}
	}
</style>
