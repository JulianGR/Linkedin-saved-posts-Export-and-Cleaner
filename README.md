# Linkedin-saved-posts-Export-and-Cleaner

Two browser console scripts to export and bulk-unsave your LinkedIn saved posts.

LinkedIn provides no native way to export or bulk-delete saved items. These scripts run in the DevTools console, no extensions or third-party services involved.

## Usage

1. Go to `https://www.linkedin.com/my-items/saved-posts/`.
2. Open DevTools (F12) and switch to the Console tab.
3. Paste the desired script and press Enter.

## 1. Export saved post URLs

Scrolls through the entire saved posts list, collects every post URL, deduplicates them, and downloads a `.txt` file. Also copies the list to the clipboard.

```javascript
(async () => {
  const sleep = ms => new Promise(r => setTimeout(r, ms));

  let prev = 0, stable = 0;
  while (stable < 4) {
    window.scrollTo(0, document.body.scrollHeight);
    await sleep(2500);
    const h = document.body.scrollHeight;
    if (h === prev) stable++;
    else { stable = 0; prev = h; }
  }

  const anchors = document.querySelectorAll(
    'a[href*="/feed/update/"], a[href*="/posts/"], a[href*="/pulse/"]'
  );

  const urls = [...new Set(
    [...anchors].map(a => {
      try {
        const u = new URL(a.href);
        return u.origin + u.pathname;
      } catch { return null; }
    }).filter(Boolean)
  )];

  console.log('Total URLs:', urls.length);
  console.table(urls);

  const blob = new Blob([urls.join('\n')], { type: 'text/plain' });
  const dl = document.createElement('a');
  dl.href = URL.createObjectURL(blob);
  dl.download = 'linkedin-saved-posts.txt';
  document.body.appendChild(dl);
  dl.click();
  dl.remove();

  try { await navigator.clipboard.writeText(urls.join('\n')); } catch {}

  return urls;
})();
```

## 2. Bulk unsave all posts

Iterates over every saved post, opens its action menu, and clicks Unsave. Hides the dropdown and toast notifications so the UI does not flicker. Stops automatically when the empty state appears.

Run this only after exporting the URLs. The action cannot be undone.

```javascript
(async () => {
  const sleep = ms => new Promise(r => setTimeout(r, ms));

  const style = document.createElement('style');
  style.id = 'unsave-hide';
  style.textContent = `
    .artdeco-dropdown__content--is-open { opacity: 0 !important; pointer-events: none !important; }
    .artdeco-toast-item, .artdeco-toasts { display: none !important; }
  `;
  document.head.appendChild(style);

  const realClick = (el) => {
    const rect = el.getBoundingClientRect();
    const x = rect.left + rect.width / 2;
    const y = rect.top + rect.height / 2;
    const opts = { bubbles: true, cancelable: true, view: window, clientX: x, clientY: y, button: 0 };
    el.dispatchEvent(new PointerEvent('pointerdown', opts));
    el.dispatchEvent(new MouseEvent('mousedown', opts));
    el.dispatchEvent(new PointerEvent('pointerup', opts));
    el.dispatchEvent(new MouseEvent('mouseup', opts));
    el.dispatchEvent(new MouseEvent('click', opts));
  };

  const tried = new WeakSet();

  const getTriggers = () => [...document.querySelectorAll('button.artdeco-dropdown__trigger')]
    .filter(b => {
      const l = (b.getAttribute('aria-label') || '').toLowerCase();
      return l.includes('more actions on') && !tried.has(b);
    });

  const findUnsaveItem = () => {
    const open = document.querySelector('.artdeco-dropdown__content--is-open');
    if (!open) return null;
    return [...open.querySelectorAll('div[role="button"].artdeco-dropdown__item')]
      .find(el => {
        const t = (el.textContent || '').trim().toLowerCase();
        return /^unsave\b|^quitar de elementos guardados|^remove from saved/.test(t);
      });
  };

  const forceLoadMore = async () => {
    const triggers = document.querySelectorAll('button.artdeco-dropdown__trigger[aria-label*="more actions on" i]');
    const last = triggers[triggers.length - 1];
    if (last) last.scrollIntoView({ block: 'end', behavior: 'instant' });
    window.scrollTo(0, document.body.scrollHeight);
    await sleep(3500);
  };

  const isListEmpty = () => {
    const txt = (document.body.innerText || '').toLowerCase();
    return /start saving posts|saved posts will show up here/.test(txt);
  };

  let count = 0, scrollMisses = 0;

  while (scrollMisses < 6) {
    if (isListEmpty()) {
      console.log('Empty state detected, stopping.');
      break;
    }

    let pool = getTriggers();
    console.log('Iteration: available triggers =', pool.length, '| Total unsaved =', count);

    if (pool.length === 0) {
      await forceLoadMore();
      pool = getTriggers();
      if (pool.length === 0) {
        scrollMisses++;
        console.log('Miss', scrollMisses, '/ 6');
        continue;
      }
      scrollMisses = 0;
    }

    for (const trigger of pool) {
      tried.add(trigger);
      realClick(trigger);
      await sleep(300);

      const unsave = findUnsaveItem();
      if (!unsave) {
        document.body.dispatchEvent(new KeyboardEvent('keydown', { key: 'Escape', bubbles: true }));
        await sleep(150);
        continue;
      }
      realClick(unsave);
      count++;
      if (count % 5 === 0) console.log('Unsaved:', count);
      await sleep(500);
    }
  }

  document.getElementById('unsave-hide')?.remove();
  console.log('Finished. Total unsaved:', count);
})();
```
