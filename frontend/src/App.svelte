<script>
  import FontList from "./FontList.svelte";

  function getHashCode(string) {
    let hash = 0,
      i,
      chr;
    for (i = 0; i < string.length; i++) {
      chr = string.charCodeAt(i);
      hash = (hash << 5) - hash + chr;
      hash |= 0;
    }
    return hash;
  }

  const fontRuleRegex = /@font-face\s*{([\s\S]*?)}/g;
  const declarationsRegex = /(?<property>\S*?):\s*(?<value>[\s\S]+?)(?:;|$)/g;

  let mySidebar;
  let externalCSSResources = {};
  let inlineCSSResources = {};
  let foundDeclarations = {};

  function callback() {
    chrome.devtools.inspectedWindow.eval(
      `values({ elements: $$('style').map(el => el.textContent), external: $$('link[rel="stylesheet"]').map(link => link.href) })`,
      function ([elements, external], isException) {
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
                  const fonts = [];
                  for (const [, declaration] of content.matchAll(
                    fontRuleRegex
                  )) {
                    const declarations = {};
                    for (const match of declaration.matchAll(
                      declarationsRegex
                    )) {
                      declarations[match.groups.property] = match.groups.value;
                    }
                    fonts[fonts.length] = declarations;
                  }

                  if (fonts.length > 0) {
                    foundDeclarations[stylesheet] = fonts;
                    foundDeclarations = foundDeclarations;
                  }
                });
            }

            if (
              elements.some((style) => {
                const hash = getHashCode(style);

                return !(hash in inlineCSSResources);
              })
            ) {
              for (let style of elements) {
                const hash = getHashCode(style);

                if (!(hash in inlineCSSResources)) {
                  inlineCSSResources[hash] = style;
                }
              }

              const inlineStyles = [];
              const fonts = [];

              for (let style of elements) {
                for (const [, declaration] of style.matchAll(fontRuleRegex)) {
                  const declarations = {};
                  for (const match of declaration.matchAll(declarationsRegex)) {
                    declarations[match.groups.property] = match.groups.value;
                  }
                  fonts[fonts.length] = declarations;
                }

                if (fonts.length > 0) inlineStyles[inlineStyles.length] = fonts;
              }

              if (inlineStyles.length > 0) {
                foundDeclarations["inline"] = inlineStyles;
                foundDeclarations = foundDeclarations;
              }
            }
          });
        } else {
          if (
            elements.some((style) => {
              const hash = getHashCode(style);
              return !(hash in inlineCSSResources);
            })
          ) {
            for (let style of elements) {
              const hash = getHashCode(style);

              if (!(hash in inlineCSSResources)) {
                inlineCSSResources[hash] = style;
              }
            }

            const inlineStyles = [];
            const fonts = [];

            for (let style of elements) {
              for (const [, declaration] of style.matchAll(fontRuleRegex)) {
                const declarations = {};
                for (const match of declaration.matchAll(declarationsRegex)) {
                  declarations[match.groups.property] = match.groups.value;
                }
                fonts[fonts.length] = declarations;
              }

              if (fonts.length > 0) inlineStyles[inlineStyles.length] = fonts;
            }

            if (inlineStyles.length > 0) {
              foundDeclarations["inline"] = inlineStyles;
              foundDeclarations = foundDeclarations;
            }
          }
        }
      }
    );
  }

  function getFileName(string) {
    const array = string.split("/");
    return array[array.length - 1];
  }

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

  callback();

  $: declarations = Object.entries(foundDeclarations);
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
