<!--
 Open MCT, Copyright (c) 2014-2024, United States Government
 as represented by the Administrator of the National Aeronautics and Space
 Administration. All rights reserved.

 Open MCT is licensed under the Apache License, Version 2.0 (the
 "License"); you may not use this file except in compliance with the License.
 You may obtain a copy of the License at
 http://www.apache.org/licenses/LICENSE-2.0.

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
 WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
 License for the specific language governing permissions and limitations
 under the License.

 Open MCT includes source code licensed under additional open source
 licenses. See the Open Source Licenses file (LICENSES.md) included with
 this source code distribution or the Licensing information page available
 at runtime from the About dialog for additional information.
-->

<template>
  <LayoutFrame
    :item="item"
    :grid-size="gridSize"
    :is-editing="isEditing"
    @move="move"
    @end-move="endMove"
  >
    <template #content>
      <div v-show="showImage" aria-label="Image View" class="c-image-view" :style="style"></div>
    </template>
  </LayoutFrame>
</template>

<script>
import { encode_url } from '../../../utils/encoding';
import conditionalStylesMixin from '../mixins/objectStyles-mixin.js';
import LayoutFrame from './LayoutFrame.vue';

export default {
  makeDefinition(openmct, gridSize, element) {
    return {
      stroke: 'transparent',
      x: 1,
      y: 1,
      width: 10,
      height: 5,
      url: element.url
    };
  },
  components: {
    LayoutFrame
  },
  mixins: [conditionalStylesMixin],
  inject: ['openmct'],
  props: {
    item: {
      type: Object,
      required: true
    },
    gridSize: {
      type: Array,
      required: true,
      validator: (arr) => arr && arr.length === 2 && arr.every((el) => typeof el === 'number')
    },
    index: {
      type: Number,
      required: true
    },
    initSelect: Boolean,
    isEditing: {
      type: Boolean,
      required: true
    }
  },
  emits: ['move', 'end-move'],
  computed: {
    showImage() {
      return this.isEditing || !this.itemStyle?.isStyleInvisible;
    },
    style() {
      let backgroundImage = `url('${encode_url(this.item.url)}')`;
      let border = '1px solid ' + this.item.stroke;

      if (this.itemStyle) {
        if (this.itemStyle.imageUrl !== undefined) {
          backgroundImage = `url('${encode_url(this.itemStyle.imageUrl)}')`;
        }

        border = this.itemStyle.border;
      }

      return {
        backgroundImage,
        border
      };
    }
  },
  watch: {
    index(newIndex) {
      if (!this.context) {
        return;
      }

      this.context.index = newIndex;
    },
    item(newItem) {
      if (!this.context) {
        return;
      }

      this.context.layoutItem = newItem;
    }
  },
  mounted() {
    this.context = {
      layoutItem: this.item,
      index: this.index
    };
    this.removeSelectable = this.openmct.selection.selectable(
      this.$el,
      this.context,
      this.initSelect
    );
  },
  beforeUnmount() {
    if (this.removeSelectable) {
      this.removeSelectable();
    }
  },
  methods: {
    move(gridDelta) {
      this.$emit('move', gridDelta);
    },
    endMove() {
      this.$emit('end-move');
    }
  }
};
</script>
