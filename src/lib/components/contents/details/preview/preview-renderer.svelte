<script>
  import { _ } from 'svelte-i18n';
  import { entryDraft } from '$lib/services/contents/draft';
  import { onMount } from 'svelte';

  let { previewRenderer, locale } = $props();

  let mainRef;
  let iFrameRef;

  let out = null;

  let htmlContent = $state(null);

  const getCurrentValue = () => {
    const currentValue = $entryDraft?.currentValues[locale] || {};

    const obj = {};

    for (const key in currentValue) {
      const parts = key.split('.');

      let current = obj;
      let currentKey = parts[0];

      for (let i = 1; i < parts.length - 1; i++) {
        const part = parts[i];

        if (!isNaN(+part)) {
          current[currentKey] ||= [];

          if (+part > current[currentKey].length) {
            current[currentKey].push({});
          }

          currentKey = +part;
          current = current[currentKey];
        } else {
          current[currentKey] ||= {};
          current = current[currentKey];
          currentKey = part;
        }
      }

      current[currentKey] = currentValue[key];
    }

    return { value: obj, locale };
  };

  onMount(async () => {
    out = previewRenderer(mainRef);

    const currentValue = getCurrentValue();

    if (currentValue) {
      htmlContent = await out(currentValue);
    }
  });

  $effect(() => {
    const currentValue = getCurrentValue();

    if (currentValue && out) {
      (async () => {
        htmlContent = await out(currentValue);
      })();
    }
  });

  $effect(() => {
    //const iframeDoc = iFrameRef.contentDocument || iFrameRef.contentWindow.document;

    console.log('hello', htmlContent);

    //iframeDoc.body.innerHtml = htmlContent;

    //iFrameRef.src = 'data:text/html,' + encodeURIComponent(htmlContent);

    iFrameRef.srcdoc = htmlContent;
  });
</script>

<div bind:this={mainRef} style="width:100%; height:100%">
  <iframe bind:this={iFrameRef} width="100%" height="100%" class="w-full h-full"> </iframe>

  {#if false && htmlContent}
    {@html htmlContent}
  {/if}
</div>
