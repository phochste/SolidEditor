<script>
	import Login from './Login.svelte';
	import Logout from './Logout.svelte';
	import Editor from './Editor.svelte';

	export let name;
	let profile;
	let stored = false;
	let resource;

</script>

<nav class="navbar navbar-default">
	<div class="container-fluid">
	  <div class="navbar-header">
		<span class="navbar-brand">{name}</span>
	  </div>
	  <ul class="nav navbar-nav">
		<li>
			<Logout bind:profile={profile}/>
		</li>
	  </ul>
	</div>
</nav>

<Login bind:profile={profile}/>

<div class="container" style="height: 100% !important">
{#if typeof(profile) != "undefined" && profile.webId && window.location.hash }
	<Editor url={window.location.hash.substring(1)}/> 
{:else}
	<h1>Welcome to the Acme Editor</h1>
	<p>
		This is a VSCode-like editor for your Solid text files.
	</p>
	<p>
	{#if window.location.hash.substring(1)}
		Please login to edit <tt>{window.location.hash.substring(1)}</tt>.
	{:else}
		We need a URL like <tt>{location.protocol}//{location.hostname}{location.path}#https://a-resource</tt> to start editing.
	{/if}
	</p>
	<p>
		Have fun!
	</p>
	<p>
		(c) 2022 . Patrick Hochstenbach &lt;Patrick.Hochstenbach@UGent.be&gt; .
	</p>
{/if}
</div>