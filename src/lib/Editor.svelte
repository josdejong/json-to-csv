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

  $: updateCsvContents(csv)
  $: updateJsonContents(json)

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
      csv = json2csv(parsedJson)
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
</script>

<div class='csv-container' class:fullscreen={fullscreen} on:keydown={handleKeydown}>
  <div class='csv-column'>
    <div class='csv-menu'>
      <div class='csv-title'>CSV</div>
      <label title='Check "header" when the CSV data contains a header row'><input type='checkbox' bind:checked={header}>header</label>
      <select title='Delimiter' bind:value={delimiter}>
        <option value=','>comma</option>
        <option value=';'>semicolon</option>
        <option value='\t'>tab</option>
        <option value=' '>space</option>
      </select>
      <button class='csv-action' on:click={convertToJson} title="Convert CSV to JSON"
        >To JSON {'\u25B6'}</button
      >
    </div>
    <div bind:this={refCsvEditor} class='csv-editor' />
  </div>
  <div class='csv-column'>
    <div class='csv-menu'>
      <button class='csv-action' on:click={convertToCsv} title="Convert JSON to CSV"
        >{'\u25C0'} To CSV</button
      >
      <div class='csv-title csv-center'>JSON</div>
      <button class='csv-action' on:click={() => fullscreen = !fullscreen} title='Toggle full screen (ESC to exit)'>
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
      right: 0;
      bottom: 0;
      z-index: 9999;
      border: 5px solid white;
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

          &.csv-center {
            text-align: center;
          }
        }

        label,
        select,
        button {
          cursor: pointer;
        }

        .csv-action {
          background: $button-background-color;
          color: $color;
          font-family: inherit;
          font-size: inherit;
          border: none;
          padding: 5px 10px;
          margin: 5px;
          border-radius: 3px;

          &:hover {
            background: lighten($button-background-color, 10%);
          }
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
