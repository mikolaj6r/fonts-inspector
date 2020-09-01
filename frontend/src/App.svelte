<script>
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
                    for (const [, declaration] of style.matchAll(
                      fontRuleRegex
                    )) {
                      const declarations = {};
                      for (const match of declaration.matchAll(
                        declarationsRegex
                      )) {
                        declarations[match.groups.property] =
                          match.groups.value;
                      }
                      fonts[fonts.length] = declarations;
                    }

                    if (fonts.length > 0)
                      inlineStyles[inlineStyles.length] = fonts;
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
                    for (const match of declaration.matchAll(
                      declarationsRegex
                    )) {
                      declarations[match.groups.property] = match.groups.value;
                    }
                    fonts[fonts.length] = declarations;
                  }

                  if (fonts.length > 0)
                    inlineStyles[inlineStyles.length] = fonts;
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
    const { index } = event.target.dataset;
    chrome.devtools.inspectedWindow.eval(`inspect($$('style')[${index}])`);
  }

  function externalSourceClicked(event) {
    const { url } = event.target.dataset;
    chrome.devtools.panels.openResource(url);
  }

  callback();
</script>

<main>
  {#if Object.keys(foundDeclarations).length > 0}
    <ul>
      {#each Object.entries(foundDeclarations) as [source, values], i}
        <li>
          {#if source == 'inline'}
            {#each values as style, j}
              <ul>
                {#each style as font, k}
                  <li class="font-list">
                    font {'{'}<span
                      class="inline-source-link"
                      on:click={inlineSourceClicked}
                      data-index={j}>style element</span>
                    <ul class="font">
                      {#each Object.entries(font) as [property, value], l}
                        <li>
                          <span class="property">{property}</span><span
                            class="value-separator">:</span>{value};
                        </li>
                      {/each}
                    </ul>
                    {'}'}
                  </li>
                {/each}
              </ul>
            {/each}
          {:else}
            <ul>
              {#each values as font, j}
                <li class="font-list">
                  font {'{'}
                  <span
                    class="inline-source-link"
                    on:click={externalSourceClicked}
                    data-url={source}>{getFileName(source)}</span>
                  <ul class="font">
                    {#each Object.entries(font) as [property, value], k}
                      <li>
                        <span class="property">{property}</span><span
                          class="value-separator">:</span>{value};
                      </li>
                    {/each}
                  </ul>
                  {'}'}
                </li>
              {/each}
            </ul>
          {/if}
        </li>
      {/each}
    </ul>
  {:else}
  No registered fonts found.
  {/if}
</main>


<style>
  main {
    text-align: left;
  }

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