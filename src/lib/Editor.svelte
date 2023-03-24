<script lang="ts">
  import { basicSetup, EditorView } from 'codemirror'
  import { EditorState } from '@codemirror/state'
  import { keymap, ViewUpdate } from '@codemirror/view'
  import { json as jsonLang } from '@codemirror/lang-json'
  import { indentWithTab, defaultKeymap } from '@codemirror/commands'
  import { lintGutter } from '@codemirror/lint'

  import { onMount } from 'svelte'
  import { csv2json, json2csv } from 'csv42'

  let refCsvEditor
  let refJsonEditor
  let csvEditorView: EditorView
  let jsonEditorView: EditorView

  export let csv: string = ''
  export let json: string = ''

  let header = true
  let delimiter = ','
  let fullscreen = false
  let showOptions = false

  let editorRef
  let originalParentRef

  $: updateCsvContents(csv)
  $: updateJsonContents(json)
  $: applyFullscreen(fullscreen)

  convertToJson()

  onMount(() => {
    createCsvEditor(refCsvEditor)
    createJsonEditor(refJsonEditor)
  })

  function createCsvEditor(target: HTMLDivElement) {
    let startState = EditorState.create({
      doc: csv,
      extensions: [
        keymap.of([...defaultKeymap, indentWithTab]),
        lintGutter(),
        basicSetup,
        // EditorView.lineWrapping,
        EditorView.updateListener.of((update: ViewUpdate) => {
          if (update.docChanged) {
            console.log('update csv')
            csv = update.state.doc.toString()
          }
        })
      ]
    })

    csvEditorView = new EditorView({
      state: startState,
      parent: target
    })
  }

  function createJsonEditor(target: HTMLDivElement) {
    let startState = EditorState.create({
      doc: json,
      extensions: [
        keymap.of([...defaultKeymap, indentWithTab]),
        jsonLang(),
        lintGutter(),
        basicSetup,
        // EditorView.lineWrapping,
        EditorView.updateListener.of((update: ViewUpdate) => {
          if (update.docChanged) {
            console.log('update json')
            json = update.state.doc.toString()
          }
        })
      ]
    })

    jsonEditorView = new EditorView({
      state: startState,
      parent: target
    })
  }

  function convertToJson() {
    try {
      const parsedJson = csv2json(csv, { header, delimiter })
      json = JSON.stringify(parsedJson, null, 2)
    } catch (err) {
      json = err.toString()
    }
  }

  function convertToCsv() {
    try {
      const parsedJson = JSON.parse(json)
      csv = json2csv(parsedJson, { header, delimiter })
    } catch (err) {
      csv = err.toString()
    }
  }

  function updateCsvContents(updatedCsv: string) {
    updateDocIfChanged(csvEditorView, updatedCsv)
  }

  function updateJsonContents(updatedJson: string) {
    updateDocIfChanged(jsonEditorView, updatedJson)
  }

  function updateDocIfChanged(editorView: EditorView, text: string) {
    if (!editorView) {
      return
    }

    const changed = text !== editorView.state.doc.toString()
    if (changed) {
      editorView.dispatch({
        changes: {
          from: 0,
          to: editorView.state.doc.length,
          insert: text
        }
      })
    }
  }

  function handleKeydown(event: KeyboardEvent) {
    if (event.key === 'Escape' && fullscreen) {
      fullscreen = false
    }
  }

  function applyFullscreen(fullscreen: boolean) {
    if (fullscreen) {
      if (editorRef) {
        originalParentRef = editorRef.parentNode
        originalParentRef.removeChild(editorRef)
        document.body.appendChild(editorRef)
      }
    } else {
      if (editorRef && originalParentRef) {
        document.body.removeChild(editorRef)
        originalParentRef.appendChild(editorRef)
      }
    }
  }
</script>

<svelte:window on:keydown={handleKeydown} />

<div class='csv-container' class:fullscreen={fullscreen} bind:this={editorRef}>
  <div class='csv-column'>
    <div class='csv-menu'>
      <button class='csv-menu-button' on:click={() => showOptions = !showOptions} title='Toggle options'>
        Options
      </button>
      <div class='csv-title'>CSV</div>
      <button class='csv-menu-button' on:click={convertToJson} title="Convert CSV to JSON"
        >To JSON {'\u25B6'}</button
      >
    </div>
    {#if showOptions}
      <div class='csv-menu csv-secondary-menu'>
        <span>Options:</span>
        <label title='Check "header" when the CSV data contains a header row'><input type='checkbox' bind:checked={header}>header</label>
        <select title='Delimiter' bind:value={delimiter}>
          <option value=','>comma</option>
          <option value=';'>semicolon</option>
          <option value='\t'>tab</option>
          <option value=' '>space</option>
        </select>
      </div>
    {/if}
    <div bind:this={refCsvEditor} class='csv-editor' />
  </div>
  <div class='csv-column'>
    <div class='csv-menu'>
      <button class='csv-menu-button' on:click={convertToCsv} title="Convert JSON to CSV"
        >{'\u25C0'} To CSV</button
      >
      <div class='csv-title'>JSON</div>
      <button class='csv-menu-button' on:click={() => fullscreen = !fullscreen} title='Toggle full screen (ESC to exit)'>
        {fullscreen ? 'Exit full screen (ESC)' : 'Full screen'}
      </button>
    </div>
    <div bind:this={refJsonEditor} class='csv-editor' />
  </div>
</div>

<style lang="scss">
  $margin: 10px;
  $color: #fff;
  $button-background-color: dodgerblue;
  $menu-background-color: #eaeaea;
  $font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen-Sans, Ubuntu;
  $font-size: 11pt;

  .csv-container {
    flex: 1;
    display: flex;
    gap: $margin;
    overflow: hidden;
    background: white;

    &.fullscreen {
      position: fixed;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      max-width: unset;
      box-sizing: border-box;
      border: 10px solid white;
      z-index: 9999;
    }

    .csv-column {
      flex: 1;
      display: flex;
      flex-direction: column;
      overflow:hidden;

      .csv-menu {
        display: flex;
        background: $menu-background-color;
        font-family: $font-family;
        font-size: $font-size;
        flex-wrap: wrap;
        align-items: center;
        gap: $margin;

        .csv-title {
          flex: 1;
          padding: 5px 10px;
          text-align: center;
        }

        label,
        select,
        button {
          cursor: pointer;
        }

        > * {
          margin: 5px;
        }

        .csv-menu-button {
          background: $button-background-color;
          color: $color;
          font-family: inherit;
          font-size: inherit;
          border: none;
          padding: 5px 10px;
          border-radius: 3px;

          &:hover {
            background: lighten($button-background-color, 10%);
          }
        }

        &.csv-secondary-menu {
          background: #dfdfdf;
        }
      }

      .csv-editor {
        flex: 1;
        display: flex;
        min-height: 0;
        border: 1px solid #e5e5e5;
        overflow:hidden;

        :global(.cm-editor) {
          width: 100%;
          height: 100%;
          overflow: hidden;
        }
      }
    }
  }
</style>
