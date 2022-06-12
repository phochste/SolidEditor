<script>
    import * as monaco from 'monaco-editor';
    let mimetype = require('mimetype');

    export let url;
    let contentType;
    let etag;
    let editor;
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
            const response = await solidClientAuthentication.fetch(url);

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
            console.log(error);

            status = {
                code: ')' ,
                message: error
            };

            return null;
        }
    }

    function mimeGuesser(url) {
        mimetype.set('.rdf', 'text/rdf'); 
        mimetype.set('.ttl', 'text/turtle');
        mimetype.set('.n3', 'text/n3');
        mimetype.set('.jsonld', 'application/ld+json');

        return mimetype.lookup(url);
    }

    function isSuccessfulStatusCode(statusCode) {
        return Math.floor(statusCode / 100) === 2;
    }

    function handleLanguage() {
        monaco.editor.setModelLanguage(editor.getModel(), language);
    }

    async function saveValue() {
        const value = editor.getValue();

        console.log(`updating etag: ${etag}`);

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
        
    async function update(url) {
        let data = await getResource(url);

        if (!data) {
            data = "";
        }

        editor.setValue(data);

        window.location.hash = url;
    }

    async function render(url) {
        const data = await getResource(url);

        editor = monaco.editor.create(document.getElementById('container'), {
            value: data ,
            language: language
        });

        editor.addAction({
            id: 'my-save',
            label: 'Save' ,
            keybindings: [
                monaco.KeyMod.CtrlCmd |  monaco.KeyCode.KeyS
            ],
            precondition: null ,
            contextMenuGroupId: 'save' ,
            contextMenuOrder: 3 ,
            run: function(ed) {
                saveValue();
            }
         });
    }
</script>
<div>
    {#if status}
        {status.code} : {status.message}
    {/if}
</div>
Resource: <input type="text" bind:value={url} size=60 on:change={async () => update(url)}/>
<img src="images/reload.png" 
     alt="Reload" 
     title="Reload" 
     width="30" height="30" on:click={ () => update(url) }/>
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
