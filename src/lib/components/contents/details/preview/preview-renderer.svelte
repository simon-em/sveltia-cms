<script>
  import { _ } from 'svelte-i18n';
  import { entryDraft } from '$lib/services/contents/draft';
  import { onMount } from 'svelte';

  let { previewRenderer, locale } = $props();

  let mainRef;
  let iFrameRef;

  let out = null;

  let htmlContent = $state(null);

  const convertToNestedObject = (flatObj) => {
    const result = {};

    for (const [key, value] of Object.entries(flatObj)) {
      // Split the key by dots to get the path
      const keys = key.split('.');
      let current = result;

      // Navigate/create the nested structure
      for (let i = 0; i < keys.length - 1; i++) {
        const currentKey = keys[i];

        // Check if the next key is a number (array index)
        const nextKey = keys[i + 1];
        const isNextKeyArrayIndex = /^\d+$/.test(nextKey);

        // If current key doesn't exist, create it
        if (!(currentKey in current)) {
          current[currentKey] = isNextKeyArrayIndex ? [] : {};
        }

        current = current[currentKey];
      }

      // Set the final value
      const finalKey = keys[keys.length - 1];
      current[finalKey] = value;
    }

    return result;
  };

  async function objectUrlToDataURI(objectUrl) {
    const response = await fetch(objectUrl);
    const blob = await response.blob();
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.onloadend = () => resolve(reader.result);
      reader.onerror = reject;
      reader.readAsDataURL(blob);
    });
  }

  /**
   * Recursively replaces any 'blob:' URLs in an object or array
   * with their equivalent Data URIs.
   */
  async function replaceBlobUrlsWithDataUris(obj) {
    if (Array.isArray(obj)) {
      // Recurse into array items
      return Promise.all(obj.map((item) => replaceBlobUrlsWithDataUris(item)));
    } else if (obj && typeof obj === 'object') {
      // Recurse into object keys
      const entries = await Promise.all(
        Object.entries(obj).map(async ([key, value]) => {
          return [key, await replaceBlobUrlsWithDataUris(value)];
        }),
      );
      return Object.fromEntries(entries);
    } else if (typeof obj === 'string' && obj.startsWith('blob:')) {
      // Found a blob URL — convert to Data URI
      try {
        return await objectUrlToDataURI(obj);
      } catch (err) {
        console.warn('Failed to convert blob URL:', obj, err);
        return obj; // fallback to original
      }
    }

    // Base case — return as-is
    return obj;
  }

  const getCurrentValue = async () => {
    const currentValue = $entryDraft?.currentValues[locale] || {};

    let obj = convertToNestedObject(currentValue);

    obj = await replaceBlobUrlsWithDataUris(obj);

    return { value: obj, locale };
  };

  onMount(async () => {
    out = previewRenderer(mainRef);

    const currentValue = await getCurrentValue();

    if (currentValue) {
      htmlContent = await out(currentValue);
    }
  });

  $effect(() => {
    (async () => {
      const currentValue = await getCurrentValue();

      if (currentValue && out) {
        htmlContent = await out(currentValue);
      }
    })();
  });

  $effect(() => {
    //const iframeDoc = iFrameRef.contentDocument || iFrameRef.contentWindow.document;

    //console.log('hello', htmlContent);

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
