<script lang="ts">
  import { onMount } from 'svelte'
  import { Editor } from '$lib/index.js'

  let editorRef

  let csv = stripIndent(`name,age,city
        Emily,19,Atlanta
        Kevin,19,Atlanta
        Chris,23,New York
        Michelle,27,Los Angeles
        Sarah,31,New York
        Joe,32,New York
        Robert,45,Manhattan
        `)

  onMount(() => {
    const testEditor = new Editor({ target: editorRef, props: { csv } })
  })

  function stripIndent(text: string): string {
    return text.replace(/\n +/g, '\n')
  }
</script>

<h1>JSON to CSV converter</h1>

<div bind:this={editorRef} class="editor" />

<style lang="scss">
  .editor {
    width: 100%;
    max-width: 1000px;
    height: 300px;
    display: flex;
  }
</style>
