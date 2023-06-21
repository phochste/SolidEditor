<script>
    import * as monaco from 'monaco-editor';
    import * as md5 from 'md5';
    let mimetype = require('mimetype');

    self.MonacoEnvironment = {
	    getWorkerUrl: function (moduleId, label) {
	    	if (label === 'json') {
	    		return './json.worker.bundle.js';
	    	}
	    	if (label === 'css' || label === 'scss' || label === 'less') {
	    		return './css.worker.bundle.js';
    		}
    		if (label === 'html' || label === 'handlebars' || label === 'razor') {
    			return './html.worker.bundle.js';
    		}
    		if (label === 'typescript' || label === 'javascript') {
    			return './ts.worker.bundle.js';
    		}
        	return './editor.worker.bundle.js';
    	}
    };

    export let url;
    let contentType;
    let etag;
    let editor;
    let thisVersion;
    let hasUnsavedChanges = false;
    let status;
    let languages = ['abap', 'apex', 'azcli', 'bat', 'bicep', 'cameligo', 'clojure', 'coffee', 'cpp', 'csharp', 'csp',
'css', 'dart', 'dockerfile', 'ecl', 'elixir', 'flow9', 'fsharp', 'freemarker2', 'go', 'graphql',
'handlebars', 'hcl', 'html', 'ini', 'java', 'javascript', 'julia', 'kotlin', 'less', 'lexon',
'lua', 'liquid', 'm3', 'markdown', 'mips', 'msdax', 'mysql', 'objective-c', 'pascal', 'pascaligo',
'perl', 'pgsql', 'php', 'pla', 'postiats', 'powerquery', 'powershell', 'protobuf', 'pug', 'python',
'qsharp', 'r', 'razor', 'redis', 'redshift', 'restructuredtext', 'ruby', 'rust', 'sb', 'scala',
'scheme', 'scss', 'shell', 'solidity', 'sophia', 'sparql', 'sql', 'st', 'swift', 'systemverilog',
'tcl', 'twig', 'typescript', 'vb', 'xml', 'yaml'];
    let language = 'plaintext';

    async function getResource(url) {
        try {
            etag = null;
           
            const headers  = new Headers();
            headers.append('pragma','no-cache');
            headers.append('cache-control','no-cache');
            const init = {
                method: 'GET',
                headers: headers
            };

            const response = await solidClientAuthentication.fetch(url,init);

            console.log(`url: ${url} ; status: ${response.status}`);

            status = {
                code: response.status ,
                message: response.statusText + '.'
            }

            if (isSuccessfulStatusCode(response.status)) {
                contentType = response.headers.get('Content-Type');   
            }
            else {
                contentType = mimeGuesser(url);
            } 

            if (!isSuccessfulStatusCode(response.status))
                return null;

            etag = response.headers.get('ETag');

            console.log(`etag: ${etag}`);

            return await response.text();
        } catch (error) {
            console.log(`Whoops:`,error);

            status = {
                code: '404' ,
                message: error
            };

            return null;
        }
    }

    function mimeGuesser(url) {
        mimetype.set('.yml', 'application/x-yaml');
        mimetype.set('.rdf', 'application/rdf+xml'); 
        mimetype.set('.ttl', 'text/turtle');
        mimetype.set('.n3', 'text/n3;charset=utf-8');
        mimetype.set('.nt','application/n-triples');
        mimetype.set('.nq','application/n-quads');
        mimetype.set('.trig','application/x-trig');
        mimetype.set('.jsonld', 'application/ld+json');

        const mime = mimetype.lookup(url);

        if (mime) {
            return mime;
        }
        else {
            return "text/turtle";
        }
    }

    function isSuccessfulStatusCode(statusCode) {
        return Math.floor(statusCode / 100) === 2;
    }

    function handleLanguage() {
        monaco.editor.setModelLanguage(editor.getModel(), language);
    }

    async function saveValue(url) {
        console.log(`saveValue(${url})`);
        console.log(`etag: ${etag}`);

        const value = editor.getValue();

        try {
            const response = await solidClientAuthentication.fetch(url, {
                method: 'PUT' ,
                body: value,
                headers: {
                    "Content-Type": contentType ,
                    "If-Match": etag
                }
            });

            if (! response.ok) {
                console.error(`Request failed: ${response.status}`);

                let reason = response.statusText;

                if (0) {}
                else if (response.status == 404) {
                    reason = 'You don\'t have permissions to update this resource';                
                }
                else if (response.status == 412) {
                    reason = 'Someone else was editing this resource, please reload';
                }

                status= {
                    code: response.status ,
                    message: `Failed to save results: ${reason}.`
                }
            }
            else {
                console.log("Stored!");
                status= {
                    code: response.status ,
                    message: `Stored.`
                }

                setEtag(url);

                thisVersion = md5(value);

                console.log(`md5: ${thisVersion}`);

                hasUnsavedChanges = false;
            }
        }
        catch (error) {
            console.log(error);
        
            status = {
                code: ')' ,
                message: error
            };
        }
    }

    if (url) {
        render(url);
    }

    async function setEtag(url) {
        console.log(`setEtag(${url})`);
        try {
            const response = await solidClientAuthentication.fetch(url);
            if (response.ok) {
                etag = response.headers.get('ETag');
                console.log(`etag: ${etag}`);
            }
        }
        catch (e) {
            console.error("failed to get etag : %O", e);
        }
    }
        
    async function update(url) {
        let data = await getResource(url);

        if (!data) {
            data = "";
        }

        editor.setValue(data);

        let newUrl = location.protocol + '//' + location.host;

        if (location.pathname) {
            newUrl += location.pathname;
        }

        newUrl += '?resource=' + escape(url);

        window.history.pushState({},undefined,newUrl);
    }

    async function render(resourceUrl) {
        console.log(`rendering: ${resourceUrl}`);

        const data = await getResource(resourceUrl);

        editor = monaco.editor.create(document.getElementById('container'), {
            value: data ,
            language: language
        });

        if (data) {
            thisVersion = md5(data);
            console.log(`md5: ${thisVersion}`);
            editor.getModel().onDidChangeContent(evt => {
                const version = md5(editor.getValue());
                if (version != thisVersion) {
                 hasUnsavedChanges = true;
                }
                else {
                    hasUnsavedChanges = false;
                }
            });
        }

        editor.addAction({
            id: 'my-save',
            label: 'Save' ,
            keybindings: [
                monaco.KeyMod.CtrlCmd |  monaco.KeyCode.KeyS
            ],
            precondition: null ,
            contextMenuGroupId: 'save' ,
            contextMenuOrder: 1 ,
            run: function(ed) {
                saveValue(url);
            }
         });
        
         editor.addAction({
            id: 'my-save-as',
            label: 'Save As...',
            precondition: null ,
            contextMenuGroupId: 'save' ,
            contextMenuOrder: 2 ,
            run: function(ed) {
                let newurl = prompt("Resource URL:");
                saveValue(newurl);
                url = newurl;
            } 
         });
    }
</script>
<svelte:head>
    <title>Acme Editor - {url}</title>
</svelte:head>
<button 
    on:click={ () => { saveValue(url); } }
    type="button" 
    class="btn btn-primary"
    title="Save the resource. Use ctrl/cmd+s as shortcut."
    >Save</button>
{#if hasUnsavedChanges}
    <i>Unsaved changes...</i>
{/if}
<div>
    {#if status}
        &lt;{url}&gt; {status.code} : {status.message}
    {/if}
</div>
<img src="images/reload.png" 
     alt="Reload" 
     title="Reload" 
     width="30" height="30" on:click={ () => update(url) }/>
<input type="text" bind:value={url} size=80 on:change={async () => update(url)}/>
Content-Type: <input type="text" bind:value={contentType} />
Language:
<select bind:value={language}  on:change={handleLanguage}>
   <option>--Select--</option>
   {#each languages as lang}
    {#if lang == language}
        <option value={lang} selected>{lang}</option> 
    {:else}
        <option value={lang}>{lang}</option> 
    {/if}
   {/each}
</select>
<div id="container" style="height: 100% !important"></div>
