<script>
    import { onMount } from 'svelte';
    import * as N3 from 'n3';
    export let profile = undefined;

    let webId;
    let issuer;
    let showConnect = true;

    const onConnect = (ev) => { showConnect = false };
    const cancelConnect = (ev) => { showConnect = true };

    solidClientAuthentication.onLogin( () => sessionChanged(window.location.href));
    solidClientAuthentication.onSessionRestore( url => sessionChanged(url) );
    solidClientAuthentication.onLogout( () => {
        profile = undefined;
    });

    async function sessionChanged(url) {
        console.log("returned url: %s", url);
        let session = solidClientAuthentication.getDefaultSession();
        webId = session.info.webId;
        profile = await fetchUserProfile(webId);

        if (url) {
            let hash = url; 
            hash = unescape(hash.replace(/.*\?resource=/,''));
            console.log("restored hash: %s",hash);
            url  = url.replace(/\?.*/,'') + '#' + hash; 
            window.history.pushState({},undefined,url);
        }
    }

    function handleLogin() {
        console.log(`Login to : ${issuer}`);
        solidClientAuthentication.login({
            oidcIssuer: issuer,
            redirectUrl: window.location.href,
            clientName: "FormViewer"
        });
    }

    // From https://github.com/ewingson/nox
    async function readSolidDocument(url) {
        try {
            const response = await solidClientAuthentication.fetch(url, { headers: { Accept: 'text/turtle' } });

            if (!isSuccessfulStatusCode(response.status))
                return null;

            const data = await response.text();
            const parser = new N3.Parser({ baseIRI: url });

            return parser.parse(data);
        } catch (error) {
            return null;
        }
    }

    function isSuccessfulStatusCode(statusCode) {
        return Math.floor(statusCode / 100) === 2;
    }

    async function fetchUserProfile(webId) {
        const profileQuads = await readSolidDocument(webId);
        const givenNameQuad 
              = profileQuads.find(quad => quad.predicate.value === 'http://xmlns.com/foaf/0.1/givenName');
        const familyNameQuad 
              = profileQuads.find(quad => quad.predicate.value === 'http://xmlns.com/foaf/0.1/familyName');
        const nameQuad  
              = profileQuads.find(quad => quad.predicate.value === 'http://xmlns.com/foaf/0.1/name');
        const imageQuad 
              = profileQuads.find(quad => quad.predicate.value === 'http://xmlns.com/foaf/0.1/img');

        return {
            webId: webId ,
            givenName: givenNameQuad?.object.value ,
            familyName: familyNameQuad?.object.value ,
            name:  nameQuad?.object.value ,
            image: imageQuad?.object.value,
        };
    }

    onMount( () =>{
        let resourceUrl;

        if (window.location.hash) {
            resourceUrl = window.location.protocol + '//' + 
                          window.location.host + 
                          window.location.pathname + '?resource=' +
                          escape(window.location.hash.substring(1));
        }
        else {
            resourceUrl = window.location.href;
        }

        solidClientAuthentication.handleIncomingRedirect({ 
             restorePreviousSession: true ,
             url: resourceUrl
        });
    });
</script>

{#if ! profile}
   {#if showConnect}
      <button on:click|preventDefault={onConnect}>Login</button> 
   {:else}
   <form>
      <div class="row">
        <div class="col-sm-12">
          <label for="inputsm">Solid IDP</label>
          <input
            class="form-control input-sm"
            style="max-width: 300px; align: right"
            id="inputsm"
            type="text"
            bind:value={issuer} />
          <button 
            class="btn btn-success"
            on:click|preventDefault={handleLogin}>Log In</button>
          <button 
            class="btn btn-danger"
            on:click|preventDefault={cancelConnect}>Cancel</button> 
        </div>
      </div>
   </form>
   {/if}
{/if}