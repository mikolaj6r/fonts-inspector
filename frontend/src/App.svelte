<script>
  import FontList from "./FontList.svelte";
  import { getHashCode, getFileName, getFonts } from "./utils";

  const externalCSSResources = {};
  const inlineCSSResources = {};
  const foundDeclarations = {};
  $: declarations = Object.entries(foundDeclarations);

  function getStyles(elements, resourcesMap) {
    if (
      elements.some((style) => {
        const hash = getHashCode(style);
        return !(hash in resourcesMap);
      })
    ) {
      for (let style of elements) {
        const hash = getHashCode(style);

        if (!(hash in resourcesMap)) {
          resourcesMap[hash] = style;
        }
      }

      const inlineStyles = [];

      for (let style of elements) {
        const fonts = getFonts(style);
        if (fonts.length > 0) inlineStyles[inlineStyles.length] = fonts;
      }

      return inlineStyles;
    }
  }

  chrome.devtools.inspectedWindow.eval(
    `values({ elements: $$('style').map(el => el.textContent), external: $$('link[rel="stylesheet"]').map(link => link.href) })`,
    function ([elements, external], isException) {
      const inlineStyles = getStyles(elements, inlineCSSResources);
      if (inlineStyles.length > 0) {
        foundDeclarations["inline"] = inlineStyles;
        foundDeclarations = foundDeclarations;
      }

      if (external.some((link) => !(link in externalCSSResources))) {
        for (let link of external) {
          if (!(link in externalCSSResources)) {
            externalCSSResources[link] = link;
          }
        }

        chrome.devtools.inspectedWindow.getResources((resources) => {
          for (let stylesheet of Object.keys(externalCSSResources)) {
            resources
              .find((res) => res.url == stylesheet)
              .getContent((content) => {
                const fonts = getFonts(content);

                if (fonts.length > 0) {
                  foundDeclarations[stylesheet] = fonts;
                  foundDeclarations = foundDeclarations;
                }
              });
          }
        });
      }
    }
  );

  function inlineSourceClicked(event) {
    const { index } = event.currentTarget.dataset;
    const { action } = event.target.dataset;

    if (action !== "source-link") return;

    chrome.devtools.inspectedWindow.eval(`inspect($$('style')[${index}])`);
  }

  function externalSourceClicked(event) {
    const { url } = event.currentTarget.dataset;
    const { action } = event.target.dataset;

    if (action !== "source-link") return;

    chrome.devtools.panels.openResource(url);
  }
</script>

<style>
  main {
    text-align: left;
  }

  ul {
    margin: 0;
    padding: 0;
    list-style-type: none;
  }
</style>

<main>
  {#if declarations.length > 0}
    <ul>
      {#each declarations as [source, values]}
        <li>
          {#if source === 'inline'}
            {#each values as style, j}
              <ul on:click={inlineSourceClicked} data-index={j}>
                <FontList fonts={style} source={'style element'} />
              </ul>
            {/each}
          {:else}
            <ul on:click={externalSourceClicked} data-url={source}>
              <FontList fonts={values} source={getFileName(source)} />
            </ul>
          {/if}
        </li>
      {/each}
    </ul>
  {:else}No registered fonts found.{/if}
</main>
