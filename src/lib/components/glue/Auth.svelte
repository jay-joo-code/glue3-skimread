<script lang="ts">
	import { goto } from '$app/navigation';
	import { page } from '$app/stores';
	import { APP_NAME, IS_ENFORCE_CORNELL_EMAIL, PRIVATE_NAVS } from '$lib/glue/config';
	import { supabase } from '$lib/glue/supabaseClient';
	import IconGoogle from '$lib/icons/glue/IconGoogle.svelte';
	import IconLogout from '$lib/icons/glue/IconLogout.svelte';
	import IconPerson from '$lib/icons/glue/IconPerson.svelte';

	let email: string;
	let password: string;
	let authError: string;
	let state: 'signin' | 'register' = 'signin';
	let authMethod: 'magic-link' | 'email' | 'google' = 'magic-link';
	let magicLinkState: 'sent' | 'not-sent' = 'not-sent';
	let isAuthLoading = false;

	const signInWithGoogle = async () => {
		await supabase.auth.signInWithOAuth({
			provider: 'google'
		});
	};

	const signOut = async () => {
		await supabase.auth.signOut();
		goto('/');
	};

	const toggleState = () => {
		if (state === 'signin') state = 'register';
		else state = 'signin';
	};

	const signUpEmailPwd = async () => {
		let { data, error } = await supabase.auth.signUp({
			email,
			password
		});

		if (error) {
			authError = error;
		} else {
			goto('/');
		}
	};

	const loginEmailPwd = async () => {
		let { data, error } = await supabase.auth.signInWithPassword({
			email,
			password
		});

		if (error) {
			authError = error;
		} else {
			goto('/');
		}
	};

	const handleEmailPwdSubmit = async () => {
		if (state === 'register') {
			signUpEmailPwd();
		} else {
			loginEmailPwd();
		}
	};

	async function signInEmailMagicLink() {
		isAuthLoading = true;
		const { data, error } = await supabase.auth.signInWithOtp({
			email,
			options: {
				emailRedirectTo: `${window.location.origin}/magic-link-redirect`
			}
		});
		magicLinkState = 'sent';
		isAuthLoading = false;
	}

	console.log('$page?.data?.session', $page?.data?.session);
</script>

{#if $page?.data?.session}
	<!-- if authenticated, show avatar -->
	<div class="dropdown-end dropdown">
		<label tabindex="0" class="btn-ghost btn-square btn">
			<div class="placeholder avatar">
				<div
					class="w-8 rounded-full bg-neutral-focus text-neutral-content ring-2 ring ring-primary ring-offset-2 ring-offset-base-100"
				>
					{#if $page?.data?.session?.user?.user_metadata?.avatar_url}
						<img src={$page?.data?.session?.user?.user_metadata?.avatar_url} />
					{:else if $page?.data?.session?.user?.user_metadata?.name}
						<span class="text-sm">{$page?.data?.session?.user?.user_metadata?.name?.charAt(0)}</span
						>
					{:else}
						<div class="text-xl">
							<IconPerson />
						</div>
					{/if}
				</div>
			</div>
		</label>
		<ul
			tabindex="0"
			class="dropdown-content menu rounded-box menu-compact mt-3 w-52 bg-base-200 p-2 shadow"
		>
			{#each PRIVATE_NAVS as nav}
				<li>
					<a href={nav.path}>
						<svelte:component this={nav.icon} />{nav.label}
					</a>
				</li>
			{/each}
			<li><button on:click={signOut}><IconLogout />Logout</button></li>
		</ul>
	</div>
{:else}
	<!-- else, show sign in button -->
	<button>
		<label for="modal-auth" class="btn-ghost btn-sm btn md:btn-md">Sign in</label>
	</button>

	<!-- sign in modal -->
	<input type="checkbox" id="modal-auth" class="modal-toggle" />
	<label for="modal-auth" class="modal cursor-pointer">
		<label class="modal-box relative w-11/12 max-w-sm" for="">
			<div class="flex flex-col gap-3">
				{#if IS_ENFORCE_CORNELL_EMAIL}
					<!-- force Cornell auth -->
					<h3 class="p-0 text-xl font-medium">
						Sign in
						<span
							>with your <span class="underline decoration-primary underline-offset-2"
								>Cornell&nbsp;email</span
							></span
						>
					</h3>
					<p class="mb-2 text-sm text-base-content/80">
						Access all of the features by signing in and getting started with {APP_NAME}.
					</p>
					<p class="mb-2 text-sm text-base-content/80">
						You must sign in with your Cornell email to sign into {APP_NAME}!
					</p>
					<button type="button" class="btn-primary btn mt-2 gap-2" on:click={signInWithGoogle}
						><IconGoogle /> Sign in with {IS_ENFORCE_CORNELL_EMAIL
							? 'Cornell email'
							: 'Google'}</button
					>
				{:else if authMethod === 'magic-link'}
					{#if magicLinkState === 'not-sent'}
						<form class="pb-4" on:submit={signInEmailMagicLink}>
							<h3 class="p-0 text-xl font-bold">Get started with {APP_NAME}</h3>
							<div class="form-control mt-6">
								<label class="label font-medium text-base-content/70" for="email">Email</label>
								<input
									class="input-bordered input w-full max-w-xs"
									placeholder="Your school email"
									type="email"
									name="email"
									required
									bind:value={email}
								/>
							</div>
							<button
								class="btn-primary btn-block btn mt-6 rounded-full {isAuthLoading && 'loading'}"
								>Sign in with magic link</button
							>
						</form>
					{:else if magicLinkState === 'sent'}
						<!-- magic link sent -->
						<h3 class="p-0 text-xl font-bold">Check your email!</h3>
						<p class="text-sm text-base-content/80">
							We've sent a magic link to your email. Click the link to sign in.
						</p>
						<button
							class="btn-block btn mt-4 rounded-full {isAuthLoading && 'loading'}"
							on:click={signInEmailMagicLink}>Resend magic link</button
						>
					{/if}
				{:else if authMethod === 'email'}
					<!-- email pwd auth -->
					<form on:submit={handleEmailPwdSubmit}>
						<div class="flex flex-col gap-3">
							<h3 class="p-0 text-xl font-bold">
								{state === 'signin' ? 'Sign in' : 'Create an account'}
							</h3>
							<div class="form-control">
								<label class="label" for="email">Email</label>
								<input
									class="input-bordered input w-full max-w-xs"
									type="email"
									name="email"
									placeholder="name@company.com"
									required
									bind:value={email}
								/>
							</div>
							<div class="form-control">
								<label class="label" for="password">Password</label>
								<input
									class="input-bordered input w-full max-w-xs"
									type="password"
									name="password"
									placeholder=""
									required
									bind:value={password}
								/>
							</div>
							<div class="flex items-center justify-between">
								{#if state === 'signin'}
									<div>
										<a
											href="/"
											class="ml-auto text-sm text-blue-700 hover:underline dark:text-blue-500"
											>Lost password?</a
										>
									</div>
								{/if}
							</div>
							{#if authError}
								<p class="text-sm text-error">{authError}</p>
							{/if}
							<button type="submit" class="btn-primary btn-block btn"
								>{state === 'signin' ? 'Sign in' : 'Create account'}</button
							>
							<div class="text-sm font-medium text-gray-500 dark:text-gray-300">
								{state === 'signin' ? 'Not registered?' : 'Already have an account?'}
								<button
									class="text-blue-700 hover:underline dark:text-blue-500"
									on:click={toggleState}
									>{state === 'signin' ? 'Create account' : 'Sign in'}
								</button>
							</div>
						</div>
					</form>
				{/if}
			</div>
		</label>
	</label>
{/if}
