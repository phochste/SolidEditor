<script>
	import Login from './Login.svelte';
	import Editor from './Editor.svelte';

	export let name;
	let profile;
	let stored = false;
</script>

<nav class="navbar navbar-default">
	<div class="container-fluid">
	  <div class="navbar-header">
		<span class="navbar-brand">{name}</span>
	  </div>
	  <ul class="nav navbar-nav">
		<li>
			{#if typeof(profile) != "undefined" }
				{#if profile.image}
					<img src="{profile.image}" class="img-circle" width="50" height="50" alt="{profile.name}" title="{profile.webId}"/>
				{:else}
					<img src="images/unknown.jpeg" width="50" height="50" alt="{profile.name}" title="{profile.webId}"/>
				{/if}
			{/if}
		</li>
	  </ul>
	</div>
</nav>

<Login bind:profile={profile}/>

{#if typeof(profile) != "undefined" && profile.webId && window.location.hash }
	<Editor url={window.location.hash.substring(1)}/>
{/if}