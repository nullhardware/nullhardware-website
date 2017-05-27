Created with <https://icomoon.io>.
More documentation: <https://icomoon.io/#docs/inline-svg>.

Relevant css:

```css
.icon {
  display: inline-block;
  width: 1em;
  height: 1em;
  stroke-width: 0;
  stroke: currentColor;
  fill: currentColor;
}

/* ==========================================
Single-colored icons can be modified like so:
.icon-name {
  font-size: 32px;
  color: red;
}
========================================== */
```

Use:
```html
<svg class="icon-home">
  <use xlink:href="symbol-defs.svg#icon-home"></use>
</svg>
```

Notes: requires external svg polyfill for ie and old safari <https://github.com/Keyamoon/svgxuse>.