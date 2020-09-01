<script>
  export let fonts;
  export let source;
</script>

<style>
  .property {
    color: rgb(200 0 0);
  }

  ul {
    margin: 0;
    padding: 0;
    list-style-type: none;
  }

  .font {
    padding: 0 0 0 22px;
    font-size: 11px;
    line-height: 14px;
    font-family: dejavu sans mono, monospace;
    color: rgb(48, 57, 66);
  }

  .font-list {
    padding: 2px 2px 4px 4px;
    border-bottom: 1px solid #d0d0d0;
  }

  .value-separator {
    display: inline-block;
    width: 14px;
    text-decoration: inherit;
    white-space: pre;
  }

  .inline-source-link {
    cursor: pointer;
    float: right;
    text-align: right;
    text-decoration: underline;
    color: hsl(0deg 0% 44%);
    text-decoration-color: hsl(0deg 0% 60%);
  }
</style>

{#each fonts as font}
  <li class="font-list">
    font {'{'}<span
      class="inline-source-link"
      data-action="source-link">{source}</span>
    <ul class="font">
      {#each Object.entries(font) as [property, value]}
        <li>
          <span class="property">{property}</span><span
            class="value-separator">:</span>{value};
        </li>
      {/each}
    </ul>
    {'}'}
  </li>
{/each}
