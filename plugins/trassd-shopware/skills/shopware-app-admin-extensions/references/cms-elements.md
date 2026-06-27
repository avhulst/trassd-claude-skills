# CMS elements via the Meteor Admin SDK

Adding a CMS element to the Administration requires the Meteor Admin SDK. (Apps
can also add CMS *blocks* declaratively via `cms.xml`, but that only reuses
existing Shopware elements inside the block's slots.) Everything is loaded via
iframe; Vue 3 single-file components (SFCs) are recommended.

## File structure

```
src/Resources/app/administration/src
├── base
│   └── mainCommands.ts
├── main.ts
├── viewRenderer.ts
└── views
    └── swag-dailymotion
        ├── swag-dailymotion-config.vue
        ├── swag-dailymotion-element.vue
        └── swag-dailymotion-preview.vue
```

## Entry point — `main.ts`

Branch on the current location. The `MAIN_HIDDEN` location runs logic only (no
template); everything else renders a view.

```javascript
import { location } from '@shopware-ag/meteor-admin-sdk';

if (location.is(location.MAIN_HIDDEN)) {
    import('./base/mainCommands'); // register blocks/elements
} else {
    import('./viewRenderer');      // render Vue views
}
```

## Registering block + element — `mainCommands.ts`

```javascript
import { cms } from '@shopware-ag/meteor-admin-sdk';

const CMS_ELEMENT_NAME = 'swag-dailymotion';
export const CONSTANTS = {
    CMS_ELEMENT_NAME,
    PUBLISHING_KEY: `${CMS_ELEMENT_NAME}__config-element`,
};

// Block — appears in the block picker under the chosen category
void cms.registerCmsBlock({
    name: CONSTANTS.CMS_ELEMENT_NAME,
    label: 'Dailymotion video',
    category: 'video',
    slots: [{ element: CONSTANTS.CMS_ELEMENT_NAME }],
});

// Element — fills the block's slot
void cms.registerCmsElement({
    name: CONSTANTS.CMS_ELEMENT_NAME,
    label: 'Dailymotion video',
    defaultConfig: {
        dailyUrl: { source: 'static', value: '' },
    },
});
```

- `registerCmsElement` only → element-replacement modal only.
- `registerCmsBlock` only → block picker, but the slot renders nothing.
- Both → block picker **and** element-replacement modal. Call both for a fully
  functional addition.
- `category`: `'video'`, `'text'`, `'image'`, `'text-image'`, `'commerce'`,
  `'sidebar'`, `'form'`, or a custom string (creates a new category group).
- Best practice: keep the element name and publishing key in a constant. The
  publishing key must be the element name + the `__config-element` suffix.

## Locations and the view renderer — `viewRenderer.ts`

Registering the element auto-generates three location IDs from the element name
plus `-element`, `-config`, and `-preview`. Map each to a Vue component;
`location.get()` returns the active location ID.

```javascript
import { createApp, defineAsyncComponent, h } from 'vue';
import { location } from '@shopware-ag/meteor-admin-sdk';

location.startAutoResizer(); // signal readiness + keep iframe height in sync

const locations = {
    'swag-dailymotion-element': defineAsyncComponent(
        () => import('./views/swag-dailymotion/swag-dailymotion-element.vue')),
    'swag-dailymotion-config': defineAsyncComponent(
        () => import('./views/swag-dailymotion/swag-dailymotion-config.vue')),
    'swag-dailymotion-preview': defineAsyncComponent(
        () => import('./views/swag-dailymotion/swag-dailymotion-preview.vue')),
};

const app = createApp({ render: () => h(locations[location.get()]) });
app.mount('#app');
```

## Addressing element data

Shopware appends the current element instance id as an `elementId` query param
to the iframe URL. Combine it with the publishing key to build the data id:

```javascript
const params = new URLSearchParams(window.location.search);
const elementId = params.get('elementId');
const dataId = `${CONSTANTS.PUBLISHING_KEY}__${elementId}`;
```

## Config view — `swag-dailymotion-config.vue`

Uses the SDK `data` API: `data.get` loads current config, `data.update` persists
changes (only the changed structure, not the whole element).

```html
<template>
  <div>
    <h2>Config!</h2>
    Video-Code: <input v-model="dailyUrl" type="text"><br>
  </div>
</template>

<script setup lang="ts">
import { onBeforeMount, ref, computed } from 'vue';
import { data } from '@shopware-ag/meteor-admin-sdk';
import { CONSTANTS } from '../../base/mainCommands';

const dailyUrlValue = ref('');
const dailyUrlSource = ref('static');
const selectors = ['config.dailyUrl.value', 'config.dailyUrl.source'];

const dataId = computed(() => {
    const elementId = new URLSearchParams(window.location.search).get('elementId');
    return elementId ? `${CONSTANTS.PUBLISHING_KEY}__${elementId}` : CONSTANTS.PUBLISHING_KEY;
});

const dailyUrl = computed({
    get(): string { return dailyUrlValue.value || ''; },
    set(value: string): void {
        dailyUrlValue.value = value;
        data.update({
            id: dataId.value,
            data: { config: { dailyUrl: { value: dailyUrlValue.value, source: dailyUrlSource.value } } },
        });
    },
});

onBeforeMount(async () => {
    const value = await data.get({ id: dataId.value, selectors }) as
        { 'config.dailyUrl.value': string; 'config.dailyUrl.source': string };
    if (value) {
        dailyUrlValue.value = value['config.dailyUrl.value'];
        dailyUrlSource.value = value['config.dailyUrl.source'];
    }
});
</script>
```

- `data.get()` accepts an optional `selectors` array; the result is a flat object
  keyed by selector path (e.g. `value['config.dailyUrl.value']`).
- `data.update({ id, data })` persists only the changed config.

## Element view — `swag-dailymotion-element.vue`

Renders the element in the layout editor and stays in sync via `data.subscribe`,
which receives the same flat selector-keyed object regardless of where the change
originates.

```html
<template>
  <div class="sw-cms-el-dailymotion">
    <div class="sw-cms-el-dailymotion-iframe-wrapper">
      <iframe frameborder="0" type="text/html" width="100%" height="100%" :src="dailyUrl" />
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onBeforeMount } from 'vue';
import { data } from '@shopware-ag/meteor-admin-sdk';
import { CONSTANTS } from '../../base/mainCommands';

const dailyUrlValue = ref('');
const dailyUrlSource = ref('static');
const selectors = ['config.dailyUrl.value', 'config.dailyUrl.source'];

const dailyUrl = computed(() => {
    const code = dailyUrlValue.value || 'x8hc5d6';
    return `https://www.dailymotion.com/embed/video/${code}`;
});

const dataId = computed(() => {
    const elementId = new URLSearchParams(window.location.search).get('elementId');
    return elementId ? `${CONSTANTS.PUBLISHING_KEY}__${elementId}` : CONSTANTS.PUBLISHING_KEY;
});

onBeforeMount(async () => {
    const value = await data.get({ id: dataId.value, selectors }) as
        { 'config.dailyUrl.value': string; 'config.dailyUrl.source': string };
    if (value) {
        dailyUrlValue.value = value['config.dailyUrl.value'];
        dailyUrlSource.value = value['config.dailyUrl.source'];
    }

    data.subscribe(dataId.value, (response) => {
        const d = response.data as { 'config.dailyUrl.value': string; 'config.dailyUrl.source': string };
        dailyUrlValue.value = d['config.dailyUrl.value'];
        dailyUrlSource.value = d['config.dailyUrl.source'];
    }, { selectors });
});
</script>

<style scoped>
.sw-cms-el-dailymotion-iframe-wrapper { height: 500px; }
</style>
```

## Preview view — `swag-dailymotion-preview.vue`

Thumbnail shown in the block picker. Usually minimal — a static image, skeleton,
or logo.

```html
<template>
  <h2>Preview!</h2>
</template>
```

## Storefront representation

The element also needs a Storefront template, following the path pattern
`<app-name>/Resources/views/storefront/element/<elementname>.html.twig`, which
renders the saved config (e.g. `element.config.dailyUrl.value`):

```twig
{% block element_swag_dailymotion %}
<div class="cms-element-swag-dailymotion" style="height: 100%; width: 100%">
    {% block element_dailymotion_image_inner %}
    <div class="cms-el-swag-dailymotion">
        <div style="position:relative; padding-bottom:56.25%; height:0; overflow:hidden;">
            <iframe style="width:100%; height:100%; position:absolute; left:0; top:0; overflow:hidden"
                    src="https://www.dailymotion.com/embed/video/{{ element.config.dailyUrl.value }}"
                    frameborder="0" type="text/html" width="100%" height="100%"></iframe>
        </div>
    </div>
    {% endblock %}
</div>
{% endblock %}
```
