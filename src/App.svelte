<script>
	import Login from './Login.svelte';
	import Logout from './Logout.svelte';
	import Editor from './Editor.svelte';
	import { onMount } from 'svelte';

	export let name;
	export let resource = null;
	let profile;

	readResource(window.location.search);

	solidClientAuthentication.onSessionRestore( (url) => {
		console.log("onSessionRestore(%s)", url);
		readResource(url.replace(/.*\?/,''));
	});

	function readResource(search) {
		let params = new URLSearchParams(search);
		resource = params.get("resource");
		console.log("readResource(%s) > %s", search, resource);
	}

	function updateResource(e) {
		resource = e.target.value;
		let newUrl = location.protocol + '//' + location.host;

		if (location.pathname) {
			newUrl += location.pathname;
		}

		newUrl += '?resource=' + escape(resource);

		window.history.pushState({},undefined,newUrl);

		console.log("updateResource() > %s", resource);
	}

	onMount( () => {
		readResource(window.location.search);
	});
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
<<<<<<< HEAD
{#if typeof(profile) != "undefined" && profile.webId && resource}
	<Editor url={resource}/>
=======
{#if typeof(profile) != "undefined" && profile.webId && window.location.hash }
	<Editor url={window.location.hash.substring(1)}/> 
>>>>>>> b67cd0a74f6eb69b938f897ed568144add961e49
{:else}
	<h1>Welcome to the Acme Editor</h1>
	<p>
		This is a VSCode-like editor for your Solid text files.
	</p>
	<p>
	{#if resource}
		Please login to edit <tt>{resource}</tt>.
	{:else}
		Resource URL:
		<input type="text" size="80" value={resource}
			   on:change={updateResource}/>
	{/if}
	</p>
{/if}
	<footer class="page-footer font-small blue pt-4">
		<div class="footer-copyright text-center py-3">© 2022 Copyright:
			Patrick Hochstenbach &lt;Patrick.Hochstenbach@UGent.be&gt; .
		</div>
	</footer>
</div>

