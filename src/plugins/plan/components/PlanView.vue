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
  <div ref="plan" class="c-plan c-timeline-holder">
    <template v-if="viewBounds && !options.compact">
      <SwimLane>
        <template #label>{{ timeSystem.name }}</template>
        <template #object>
          <TimelineAxis
            :bounds="viewBounds"
            :time-system="timeSystem"
            :content-height="height"
            :rendering-engine="renderingEngine"
          />
        </template>
      </SwimLane>
    </template>
    <div class="c-plan__contents u-contents">
      <ActivityTimeline
        v-for="(group, index) in visibleActivityGroups"
        :key="`activityGroup-${group.heading}-${index}`"
        :activities="group.activities"
        :clip-activity-names="clipActivityNames"
        :heading="group.heading"
        :height="group.height"
        :row-height="rowHeight"
        :width="group.width"
        :is-nested="options.isChildObject"
        :status="status"
        @activity-selected="selectActivity"
      />
    </div>
  </div>
</template>

<script>
import { scaleLinear, scaleUtc } from 'd3-scale';

import SwimLane from '@/ui/components/swim-lane/SwimLane.vue';

import TimelineAxis from '../../../ui/components/TimeSystemAxis.vue';
import PlanViewConfiguration from '../PlanViewConfiguration.js';
import { getContrastingColor, getValidatedData, getValidatedGroups } from '../util.js';
import ActivityTimeline from './ActivityTimeline.vue';

const PADDING = 1;
const OUTER_TEXT_PADDING = 12;
const INNER_TEXT_PADDING = 15;
const TEXT_LEFT_PADDING = 5;
const ROW_PADDING = 5;
const SWIMLANE_PADDING = 3;
const ROW_HEIGHT = 22;
const MAX_TEXT_WIDTH = 300;
const MIN_ACTIVITY_WIDTH = 2;
const DEFAULT_COLOR = '#999';

export default {
  components: {
    TimelineAxis,
    SwimLane,
    ActivityTimeline
  },
  inject: ['openmct', 'domainObject', 'path'],
  props: {
    options: {
      type: Object,
      default() {
        return {
          compact: false,
          isChildObject: false
        };
      }
    },
    renderingEngine: {
      type: String,
      default() {
        return 'svg';
      }
    }
  },
  data() {
    return {
      activityGroups: [],
      viewBounds: null,
      timeSystem: null,
      planData: {},
      swimlaneVisibility: {},
      clipActivityNames: false,
      height: 0,
      rowHeight: ROW_HEIGHT
    };
  },
  computed: {
    visibleActivityGroups() {
      if (this.domainObject.type === 'plan') {
        return this.activityGroups;
      } else {
        return this.activityGroups.filter(
          (group) => this.swimlaneVisibility[group.heading] === true
        );
      }
    }
  },
  watch: {
    clipActivityNames() {
      this.setScaleAndGenerateActivities();
    }
  },
  mounted() {
    this.composition = this.openmct.composition.get(this.domainObject);
    this.planViewConfiguration = new PlanViewConfiguration(this.domainObject, this.openmct);
    this.configuration = this.planViewConfiguration.getConfiguration();
    this.isNested = this.options.isChildObject;
    this.swimlaneVisibility = this.configuration.swimlaneVisibility;
    this.clipActivityNames = this.configuration.clipActivityNames;
    // This view is used for both gantt-chart and plan domain objects
    if (this.domainObject.type === 'plan') {
      this.setupPlan(this.domainObject);
    }

    const canvas = document.createElement('canvas');
    this.canvasContext = canvas.getContext('2d');
    this.setDimensions();
    this.setTimeContext();
    this.handleConfigurationChange(this.configuration);
    this.planViewConfiguration.on('change', this.handleConfigurationChange);
    this.loadComposition();

    this.resizeObserver = new ResizeObserver(this.resize);
    this.resizeObserver.observe(this.$refs.plan);
  },
  beforeUnmount() {
    this.resizeObserver.disconnect();
    this.stopFollowingTimeContext();
    if (this.unlisten) {
      this.unlisten();
    }

    if (this.removeStatusListener) {
      this.removeStatusListener();
    }

    if (this.composition) {
      this.composition.off('add', this.handleCompositionAdd);
      this.composition.off('remove', this.handleCompositionRemove);
    }

    this.planViewConfiguration.off('change', this.handleConfigurationChange);
    if (this.stopObservingPlanChanges) {
      this.stopObservingPlanChanges();
    }
    this.planViewConfiguration.destroy();
  },
  methods: {
    setupPlan(domainObject) {
      this.planObject = domainObject;
      this.applyChangesForPlanObject(domainObject);
      this.stopObservingPlanChanges = this.openmct.objects.observe(
        domainObject,
        '*',
        this.applyChangesForPlanObject
      );
      this.removeStatusListener = this.openmct.status.observe(
        domainObject.identifier,
        this.setPlanStatus
      );
    },
    setPlanData(domainObject) {
      this.planData = getValidatedData(domainObject);
    },
    activityNameFitsRect(activityName, rectWidth) {
      return this.getTextWidth(activityName) + TEXT_LEFT_PADDING < rectWidth;
    },
    setTimeContext() {
      this.stopFollowingTimeContext();
      this.timeContext = this.openmct.time.getContextForView(this.path);
      this.followTimeContext();
    },
    followTimeContext() {
      this.updateViewBounds(this.timeContext.getBounds());

      this.timeContext.on('timeSystem', this.setScaleAndGenerateActivities);
      this.timeContext.on('boundsChanged', this.updateViewBounds);
    },
    loadComposition() {
      if (this.composition) {
        this.composition.on('add', this.handleCompositionAdd);
        this.composition.on('remove', this.handleCompositionRemove);
        this.composition.load();
      }
    },
    stopFollowingTimeContext() {
      if (this.timeContext) {
        this.timeContext.off('timeSystem', this.setScaleAndGenerateActivities);
        this.timeContext.off('boundsChanged', this.updateViewBounds);
      }
    },
    showReplacePlanDialog(domainObject) {
      const dialog = this.openmct.overlays.dialog({
        iconClass: 'alert',
        message: 'This action will replace the current Plan. Do you want to continue?',
        buttons: [
          {
            label: 'Ok',
            emphasis: true,
            callback: () => {
              this.removeFromComposition(this.planObject);
              this.setupPlan(domainObject);
              dialog.dismiss();
            }
          },
          {
            label: 'Cancel',
            callback: () => {
              this.removeFromComposition(domainObject);
              dialog.dismiss();
            }
          }
        ]
      });
    },
    handleCompositionAdd(domainObject) {
      if (this.planObject) {
        this.showReplacePlanDialog(domainObject);
      } else {
        this.setupPlan(domainObject);
        this.swimlaneVisibility = this.configuration.swimlaneVisibility;
      }
    },
    handleConfigurationChange(newConfiguration) {
      this.configuration = this.planViewConfiguration.getConfiguration();
      Object.keys(newConfiguration).forEach((key) => {
        this[key] = newConfiguration[key];
      });
    },
    handleCompositionRemove(identifier) {
      if (
        this.planObject &&
        this.openmct.objects.areIdsEqual(identifier, this.planObject?.identifier)
      ) {
        this.planObject = null;
        this.planData = {};
        this.planViewConfiguration.resetSwimlaneVisibility();
      }

      this.setScaleAndGenerateActivities();
    },
    applyChangesForPlanObject(domainObject) {
      const planDomainObject = domainObject || this.domainObject;
      this.setPlanData(planDomainObject);
      this.setPlanStatus(this.openmct.status.get(planDomainObject.identifier));
      this.setScaleAndGenerateActivities();
    },
    removeFromComposition(domainObject) {
      this.composition.remove(domainObject);
    },
    resize() {
      let clientWidth = this.getClientWidth();
      let clientHeight = this.getClientHeight();
      if (clientWidth !== this.width) {
        this.setDimensions();
        this.updateViewBounds();
      }

      if (clientHeight !== this.height) {
        this.setDimensions();
      }
    },
    getClientWidth() {
      let clientWidth = this.$refs.plan.clientWidth;

      if (!clientWidth) {
        //this is a hack - need a better way to find the parent of this component
        let parent = this.getParent();
        if (parent) {
          clientWidth = parent.getBoundingClientRect().width;
        }
      }

      return clientWidth - 200;
    },
    getParent() {
      //this is a hack - need a better way to find the parent of this component
      return this.$el.closest('.is-object-type-time-strip');
    },
    getClientHeight() {
      let clientHeight = this.$refs.plan.clientHeight;

      if (!clientHeight) {
        let parent = this.getParent();
        if (parent) {
          clientHeight = parent.getBoundingClientRect().height;
        }
      }

      return clientHeight;
    },
    updateViewBounds(bounds) {
      if (bounds) {
        this.viewBounds = bounds;
      }

      if (this.timeSystem === null) {
        this.timeSystem = this.openmct.time.getTimeSystem();
      }

      this.setScaleAndGenerateActivities();
    },
    setScaleAndGenerateActivities(timeSystem) {
      if (timeSystem) {
        this.timeSystem = timeSystem;
      }

      this.setScale(this.timeSystem);
      if (this.xScale) {
        this.generateActivities();
      }
    },
    setDimensions() {
      this.width = this.getClientWidth();
      this.height = this.getClientHeight();
    },
    setScale(timeSystem) {
      if (!this.width) {
        return;
      }

      if (!timeSystem) {
        timeSystem = this.openmct.time.getTimeSystem();
      }

      if (timeSystem.isUTCBased) {
        this.xScale = scaleUtc();
        this.xScale.domain([new Date(this.viewBounds.start), new Date(this.viewBounds.end)]);
      } else {
        this.xScale = scaleLinear();
        this.xScale.domain([this.viewBounds.start, this.viewBounds.end]);
      }

      this.xScale.range([PADDING, this.width - PADDING * 2]);
    },
    isActivityInBounds(activity) {
      return activity.start < this.viewBounds.end && activity.end > this.viewBounds.start;
    },
    /**
     * Get the width of the given text in pixels.
     * @param {string} text
     * @returns {number} width of the text in pixels (as a double)
     */
    getTextWidth(text) {
      const textMetrics = this.canvasContext.measureText(text);

      return textMetrics.width;
    },
    sortIntegerAsc(a, b) {
      const numA = parseInt(a, 10);
      const numB = parseInt(b, 10);
      if (numA > numB) {
        return 1;
      }

      if (numA < numB) {
        return -1;
      }

      return 0;
    },
    /**
     * Get the row where the next activity will land.
     * @param {number} rectX the x coordinate of the activity rect
     * @param {number} width the width of the activity rect
     * @param {Object.<string, Array.<Object>>} activitiesByRow activity arrays mapped by row value
     */
    getRowForActivity(rectX, rectWidth, activitiesByRow) {
      const sortedActivityRows = Object.keys(activitiesByRow).sort(this.sortIntegerAsc);
      let currentRow;

      function activitiesHaveOverlap(rects) {
        return rects.some((rect) => {
          const { rectStart, rectEnd } = rect;
          const calculatedEnd = rectX + rectWidth;
          const hasOverlap =
            (rectX >= rectStart && rectX <= rectEnd) ||
            (calculatedEnd >= rectStart && calculatedEnd <= rectEnd) ||
            (rectX <= rectStart && calculatedEnd >= rectEnd);

          return hasOverlap;
        });
      }

      for (let i = 0; i < sortedActivityRows.length; i++) {
        let row = sortedActivityRows[i];
        if (!activitiesHaveOverlap(activitiesByRow[row])) {
          currentRow = row;
          break;
        }
      }

      if (currentRow === undefined && sortedActivityRows.length) {
        let row = parseInt(sortedActivityRows[sortedActivityRows.length - 1], 10);
        currentRow = row + ROW_HEIGHT + ROW_PADDING;
      }

      return currentRow || SWIMLANE_PADDING;
    },
    generateActivities() {
      if (!this.planObject) {
        return;
      }
      const groupNames = getValidatedGroups(this.planObject, this.planData);

      if (!groupNames.length) {
        return;
      }

      const activityGroups = [];
      this.planViewConfiguration.initializeSwimlaneVisibility(groupNames);

      groupNames.forEach((groupName) => {
        let activitiesByRow = {};
        let currentRow = 0;

        const rawActivities = this.planData[groupName];
        if (rawActivities === undefined) {
          return;
        }

        rawActivities.forEach((rawActivity, index) => {
          if (!this.isActivityInBounds(rawActivity)) {
            return;
          }

          const currentStart = Math.max(this.viewBounds.start, rawActivity.start);
          const currentEnd = Math.min(this.viewBounds.end, rawActivity.end);
          const rectX1 = this.xScale(currentStart);
          const rectX2 = this.xScale(currentEnd);
          const rectWidth = Math.max(rectX2 - rectX1, MIN_ACTIVITY_WIDTH);

          //TODO: Fix bug for SVG where the rectWidth is not proportional to the canvas measuredWidth of the text
          const showTextInsideRect =
            this.clipActivityNames || this.activityNameFitsRect(rawActivity.name, rectWidth);
          const textStart = (showTextInsideRect ? rectX1 : rectX2) + TEXT_LEFT_PADDING;
          const color = rawActivity.color || DEFAULT_COLOR;
          let textColor = '';
          if (rawActivity.textColor) {
            textColor = rawActivity.textColor;
          } else if (showTextInsideRect) {
            textColor = getContrastingColor(color);
          }

          const textLines = this.getActivityDisplayText(
            this.canvasContext,
            rawActivity.name,
            showTextInsideRect
          );
          const textWidth = textStart + this.getTextWidth(textLines[0]) + TEXT_LEFT_PADDING;

          if (showTextInsideRect) {
            currentRow = this.getRowForActivity(rectX1, rectWidth, activitiesByRow);
          } else {
            currentRow = this.getRowForActivity(rectX1, textWidth, activitiesByRow);
          }

          let textY =
            parseInt(currentRow, 10) +
            (showTextInsideRect ? INNER_TEXT_PADDING : OUTER_TEXT_PADDING);

          if (!activitiesByRow[currentRow]) {
            activitiesByRow[currentRow] = [];
          }

          const activity = {
            color: color,
            textColor: textColor,
            exceeds: {
              start: this.xScale(this.viewBounds.start) > this.xScale(rawActivity.start),
              end: this.xScale(this.viewBounds.end) < this.xScale(rawActivity.end)
            },
            row: currentRow,
            textLines: textLines,
            textStart: textStart,
            textClass: showTextInsideRect ? '' : 'c-plan__activity-label--outside-rect',
            textY: textY,
            rectStart: rectX1,
            rectEnd: showTextInsideRect ? rectX2 : textStart + textWidth,
            rectWidth: rectWidth,
            clipPathId: this.getClipPathId(groupName, rawActivity, currentRow),
            selection: {
              groupName,
              index
            }
          };
          activitiesByRow[currentRow].push(activity);
        });

        const { swimlaneHeight, swimlaneWidth } = this.getGroupDimensions(activitiesByRow);
        const activities = Array.from(Object.values(activitiesByRow)).flat();
        activityGroups.push({
          heading: groupName,
          activities,
          height: swimlaneHeight,
          width: swimlaneWidth,
          status: this.isNested ? '' : this.status
        });
      });

      this.activityGroups = activityGroups;
    },
    /**
     * Format the activity name to fit within the activity rect with a max of 2 lines
     * @param {CanvasRenderingContext2D} canvasContext
     * @param {string} activityName
     * @param {boolean} activityNameFitsRect
     */
    getActivityDisplayText(canvasContext, activityName, activityNameFitsRect) {
      // TODO: If the activity start is less than viewBounds.start then the text should be cropped on the left/should be off-screen)
      let words = activityName.split(' ');
      let line = '';
      let activityLines = [];

      for (let n = 0; n < words.length && activityLines.length <= 2; n++) {
        let tempLine = line + words[n] + ' ';
        let textMetrics = canvasContext.measureText(tempLine);
        const textWidth = textMetrics.width;
        if (!activityNameFitsRect && textWidth > MAX_TEXT_WIDTH && n > 0) {
          activityLines.push(line);
          line = words[n] + ' ';
          tempLine = line + words[n] + ' ';
        }

        line = tempLine;
      }

      return activityLines.length ? activityLines : [line];
    },
    getGroupDimensions(activityRows) {
      let swimlaneHeight = 30;
      let swimlaneWidth = this.width;

      if (!activityRows) {
        return {
          swimlaneHeight,
          swimlaneWidth
        };
      }

      const rows = Object.keys(activityRows);

      if (rows.length) {
        const lastActivityRow = rows[rows.length - 1];
        swimlaneHeight = parseInt(lastActivityRow, 10) + ROW_HEIGHT + SWIMLANE_PADDING;
        swimlaneWidth = this.width;
      }

      return {
        swimlaneHeight,
        swimlaneWidth
      };
    },
    setPlanStatus(status) {
      this.status = status;
    },
    getClipPathId(groupName, activity, row) {
      groupName = groupName.toLowerCase().replace(/ /g, '-');
      const activityName = activity.name.toLowerCase().replace(/ /g, '-');

      return `${groupName}-${activityName}-${activity.start}-${activity.end}-${row}`;
    },
    selectActivity({ event, selection }) {
      const element = event.currentTarget;
      const multiSelect = event.metaKey;
      const { groupName, index } = selection;
      const rawActivity = this.planData[groupName][index];
      this.openmct.selection.select(
        [
          {
            element: element,
            context: {
              type: 'activity',
              activity: rawActivity
            }
          },
          {
            element: this.openmct.layout.$refs.browseObject.$el,
            context: {
              item: this.domainObject,
              supportsMultiSelect: true
            }
          }
        ],
        multiSelect
      );
    }
  }
};
</script>
