<script lang="ts">
    import { fade } from 'svelte/transition'
    import { quadInOut } from 'svelte/easing'
    import Modal from '../components/Modal.svelte';
    import { getImageDimensions, openFileDialog, stringFind, uidgen } from '../utils'
    import { onMount, onDestroy } from 'svelte';
    import AnimationView from './AnimationView.svelte';
    import XmlTableView from './XMLTableView.svelte';
    import { SpriteFrameData } from '../spriteframedata';
    import { spriteframes } from '../stores';
    import { arrayBufferToBase64 } from '../b64utils';

    async function onPNGAdd(e: Event)
    {
        const target = e.target as HTMLInputElement;
        let addedPngs = [];
        for(let f of target.files)
        {
            const [ imgWidth, imgHeight ] = await getImageDimensions(f);
            const newFrame = new SpriteFrameData(f.name + '::' + uidgen.getNewId(), 'single_frame', f, null, null);
            newFrame.rect.width = imgWidth;
            newFrame.rect.height = imgHeight;
            newFrame.frameRect.frameWidth = imgWidth;
            newFrame.frameRect.frameHeight = imgHeight;
            newFrame.animationPrefix = f.name.substring(0, f.name.indexOf('.png'));

            addedPngs.push(newFrame);
        }
        spriteframes.update((prev) => [...prev, ...addedPngs]);
    }

    async function addSpritesheetAndXML(e: Event, curSpritesheet: File, curXML: File)
    {
        if(curSpritesheet == null)
        {
            // TODO: Replace alerts with a message box or something
            alert('Please select a spritesheet');
            return true;
        }
        if(curXML == null)
        {
            // TODO: Replace alerts with a message box or something
            alert('Please select an XML');
            return true;
        }

        const spshData = arrayBufferToBase64(await curSpritesheet.arrayBuffer());

        const xmlstr = await curXML.text();
        const xmlparser = new DOMParser();
        const xmldoc = xmlparser.parseFromString(xmlstr, "text/xml");
        const subtextures = xmldoc.getElementsByTagName("SubTexture");
        const addedTextures = [];
        for(let tex of subtextures)
        {
            const texname = tex.getAttribute('name');
            const animPrefixIndex = stringFind(texname, (ch) => { return !(ch >= '0' && ch <= '9') }, true);

            const newFrame = new SpriteFrameData(curSpritesheet.name + '::' + uidgen.getNewId(), 'spritesheet_frame', curSpritesheet, curXML, spshData);
            newFrame.animationPrefix = texname.substring(0, animPrefixIndex + 1);
            newFrame.rect = {
                x: parseInt(tex.getAttribute('x')),
                y: parseInt(tex.getAttribute('y')),
                width: parseInt(tex.getAttribute('width')),
                height: parseInt(tex.getAttribute('height')),
            };
            newFrame.frameRect = {
                frameX: parseInt(tex.getAttribute('frameX')) || 0,
                frameY: parseInt(tex.getAttribute('frameY')) || 0,
                frameWidth: parseInt(tex.getAttribute('frameWidth')) || newFrame.rect.width,
                frameHeight: parseInt(tex.getAttribute('frameHeight')) || newFrame.rect.height
            };

            addedTextures.push(newFrame);
        }

        spriteframes.update((prev) => [...prev, ...addedTextures]);

        curSpritesheet = null;
        curXML = null;
        return false;
    }

    function setAnimationPrefix(prefix: string)
    {
        // for each selected, change it's animation prefix
        for(let i = 0; i < $spriteframes.length; i++)
        {
            let spr = $spriteframes[i];
            if(spr.selected)
            {
                spr.animationPrefix = prefix;
                spr.selected = false;
            }
        }
        spriteframes.set($spriteframes);
    }

    $: {
        if(selectAllCheckbox != null)
        {
            selectAllCheckbox.checked = ($spriteframes.length > 0 && $spriteframes.every((sprf) => sprf.selected)) || false;
        }
    }
    
    let optsShown = false;
    let selectionOptsShown = false;
    let spritesheetXMLModalShown = false;
    let animationPrefixModalShown = false;
    let animationViewShown = false;
    let xmlViewShown = false;

    let curSpritesheet = null;
    let curXML = null;
    
    let animPrefixInput = null;
    let selectAllCheckbox = null;
    let viewOptsDiv = null;
    let selectionsOptsDiv = null;

    const onClickHandler = (evt)=>{
        if(viewOptsDiv && evt.target !== viewOptsDiv)
        {
            optsShown = false;
        }
        if(selectionsOptsDiv && evt.target !== selectionsOptsDiv)
        {
            selectionOptsShown = false;
        }
    };
    onMount(()=>{
        document.body.addEventListener('click', onClickHandler);
    });

    onDestroy(()=>{
        document.body.removeEventListener('click', onClickHandler);
    });

    function onSpritesheetAdd(e)
    {
        curSpritesheet = e.target.files[0];
    }

    function onXMLAdd(e)
    {
        curXML = e.target.files[0];
    }

    function fadeNSlide(node, options)
    {
        const fadeAnim = fade(node, options);
        return {
            duration: options.duration,
            css: fadeAnim.css
        };
    }

    function onSelectAll(value: boolean)
    {
        spriteframes.update((prev) => prev.map((sprf) => { sprf.selected = value; return sprf }));
    }

    function deleteSelection()
    {
        // backward iteration cuz indices get messed up if we don't do that
        for (let i = $spriteframes.length - 1; i >= 0; i--)
        {
            const elem = $spriteframes[i];

            if(elem.selected)
            {
                $spriteframes.splice(i, 1);
            }
        }
        spriteframes.set($spriteframes);
    }

    function cloneSelection()
    {
        for (let i = 0; i < $spriteframes.length; i++)
        {
            const elem = $spriteframes[i];
            // console.log(elem);

            if(elem.selected)
            {
                const elemClone = elem.clone();
                elemClone.selected = false;
                elemClone.sprId += '-copy' + uidgen.getNewId();
                spriteframes.update((prev) => [...prev, elemClone]);
                elem.selected = false;
            }
        }
    }
</script>

<Modal bind:showModal={spritesheetXMLModalShown}>
    <p slot="header">Select Spritesheet or XML file</p>
    <button on:click={()=>{ openFileDialog(onSpritesheetAdd, 'image/png', false) }}>Spritesheet (.png)</button>
    <button on:click={()=>{ openFileDialog(onXMLAdd, 'text/xml', false) }}>XML File (.xml)</button>
    <p>Spritesheet selected: {(curSpritesheet !== null) ? curSpritesheet.name : ''}</p>
    <p>XML selected: {(curXML !== null) ? curXML.name : ''}</p>
    <button slot="accept-btn" on:click={async (e)=>{
        let keepModal = await addSpritesheetAndXML(e, curSpritesheet, curXML);
        if(!keepModal)
            spritesheetXMLModalShown=false;
    }}>Add Spritesheet and XML</button>
</Modal>

<Modal bind:showModal={animationPrefixModalShown}>
    <p slot="header">Enter animation prefix</p>
    <input bind:this={animPrefixInput} type="text" name="anim-prefix" id="anim-prefix" value="name"/>
    <button slot="accept-btn" on:click={()=>{ setAnimationPrefix(animPrefixInput.value); animationPrefixModalShown=false }}>Set animation prefix</button>
</Modal>

<AnimationView bind:showView={animationViewShown} />
<XmlTableView bind:showView={xmlViewShown} />

<div id="controls">
    <button>Settings</button>
    <button on:click|self|stopPropagation={()=>{ optsShown = !optsShown }}>View</button>
    {#if optsShown}
        <div bind:this={viewOptsDiv} class="view-menu" id="view-opts" in:fadeNSlide="{{ duration: 100, easing: quadInOut }}" out:fadeNSlide="{{ duration: 100, easing: quadInOut }}">
            <button on:click|stopPropagation={()=>{ xmlViewShown = true; }}>View XML structure</button>
            <button on:click|stopPropagation={()=>{ animationViewShown = true; }}>View Animation</button>
        </div>
    {/if}
    <label for="select-all">
        <input bind:this={selectAllCheckbox} type="checkbox" name="select-all" id="select-all" on:change={()=>{ onSelectAll(selectAllCheckbox.checked) }} />
        Select all
    </label>
    <button on:click|stopPropagation={()=>{ selectionOptsShown = !selectionOptsShown; }}>Selection</button>
    {#if selectionOptsShown}
        <div bind:this={selectionsOptsDiv} class="view-menu" id="view-selection-opts" in:fadeNSlide="{{ duration: 100, easing: quadInOut }}" out:fadeNSlide="{{ duration: 100, easing: quadInOut }}">
            <button on:click|stopPropagation={()=>{ animationPrefixModalShown = true; }}>Set animation prefix</button>
            <button on:click|stopPropagation={deleteSelection}>Delete Selection</button>
            <button on:click|stopPropagation={cloneSelection}>Clone Selection</button>
        </div>
    {/if}
    <button on:click={()=>{ openFileDialog(onPNGAdd, 'image/png', true) }}>Add PNGs</button>
    <button on:click={()=>{ spritesheetXMLModalShown = true }}>Add Spritesheet</button>
    <button on:click={()=>{ console.log($spriteframes); }}>Generate XML</button>
</div>

<style>
    .view-menu {
        position: absolute;
        display: flex;
        flex-direction: column;
        background-color: brown;
        padding: 2px;
    }

    #view-opts {
        left: 14%;
        top: -6em;
    }

    #view-selection-opts {
        left: 43%;
        top: -8em;
    }

    #controls {
        display: grid;
        grid-template-columns: repeat(7, 1fr);
        border: 2px solid grey;
        margin: 2px;
        position: sticky;
        z-index: 5;
        bottom: 0;
        background-color: white;
    }
</style>