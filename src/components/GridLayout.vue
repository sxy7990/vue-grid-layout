<template>
    <div ref="item" class="vue-grid-layout" :style="mergedStyle">
        <slot></slot>
        <grid-item class="vue-grid-placeholder"
                   v-show="isDragging"
                   :x="placeholder.x"
                   :y="placeholder.y"
                   :w="placeholder.w"
                   :h="placeholder.h"
                   :i="placeholder.i"></grid-item>
    </div>
</template>
<style>
    .vue-grid-layout {
        position: relative;
        transition: height 200ms ease;
        background: -webkit-linear-gradient(top, transparent 39px, blue 40px), -webkit-linear-gradient(left, transparent 39px, blue 40px);
        background-size: 40px 40px;
    }
</style>
<script>
    import Vue from 'vue';
    const elementResizeDetectorMaker = require("element-resize-detector");

    import {handlebottom, handletop, handleleft, handleright, compact, handleNegative, getLayoutItem, moveElement, validateLayout, getAllCollisions} from '@/helpers/utils';

    import GridItem from './GridItem.vue'
    import {addWindowEventListener, removeWindowEventListener} from "@/helpers/DOM";

    export default {
        name: "GridLayout",
        provide() {
            return {
                eventBus: null,
                layout: this
            }
        },
        components: {
            GridItem,
        },
        props: {
            // If true, the container height swells and contracts to fit contents
            autoSize: {
                type: Boolean,
                default: true
            },
            unit: {
                type: Number,
                default: 40
            },
            maxRows: {
                type: Number,
                default: Infinity
            },
            maxCols: {
                type: Number,
                default: Infinity
            },
            isDraggable: {
                type: Boolean,
                default: true
            },
            isResizable: {
                type: Boolean,
                default: true
            },
            useCssTransforms: {
                type: Boolean,
                default: true
            },
            layout: {
                type: Array,
                required: true,
            },
            preventCollision: {
                type: Boolean,
                default: false
            },
            useStyleCursor: {
                type: Boolean,
                default: true
            }
        },
        data: function () {
            return {
                width: null,
                mergedStyle: {},
                lastLayoutLength: 0,
                isDragging: false,
                placeholder: {
                    x: 0,
                    y: 0,
                    w: 0,
                    h: 0,
                    i: -1
                },
                layouts: {},
                originalLayout: null, // store original Layout
                verticalCompact: false,
            };
        },
        created () {
            const self = this;

            // Accessible refernces of functions for removing in beforeDestroy
            self.resizeEventHandler = function(eventType, i, x, y, h, w) {
                self.resizeEvent(eventType, i, x, y, h, w);
            };

            self.dragEventHandler = function(eventType, i, x, y, h, w) {
                self.dragEvent(eventType, i, x, y, h, w);
            };

            self._provided.eventBus = new Vue();
            self.eventBus = self._provided.eventBus;
            self.eventBus.$on('resizeEvent', self.resizeEventHandler);
            self.eventBus.$on('dragEvent', self.dragEventHandler);
            
            for (let i = 0; i < self.layout.length; i++) {
                const element = self.layout[i];
                element.offsetX = 0
                element.offsetY = 0
            }
            self.$emit('layout-created', self.layout);
        },
        beforeDestroy: function(){
            //Remove listeners
            this.eventBus.$off('resizeEvent', this.resizeEventHandler);
            this.eventBus.$off('dragEvent', this.dragEventHandler);
			this.eventBus.$destroy();
            removeWindowEventListener("resize", this.onWindowResize);
            if (this.erd) {
                this.erd.uninstall(this.$refs.item);
            }
        },
        beforeMount: function() {
            this.$emit('layout-before-mount', this.layout);
        },
        mounted: function() {
            this.$emit('layout-mounted', this.layout);
            this.$nextTick(function () {
                validateLayout(this.layout);

                this.originalLayout = this.layout;
                const self = this;
                this.$nextTick(function() {
                    self.onWindowResize();

                    addWindowEventListener('resize', self.onWindowResize);

                    compact(self.layout, self.verticalCompact);
                    handleNegative(this.layout)
                    self.updatePlayground();
                    self.$emit('layout-updated', self.layout)
                    self.$nextTick(function () {
                        this.erd = elementResizeDetectorMaker({
                            strategy: "scroll", //<- For ultra performance.
                            // See https://github.com/wnr/element-resize-detector/issues/110 about callOnAdd.
                            callOnAdd: false,
                        });
                        this.erd.listenTo(self.$refs.item, function () {
                            self.onWindowResize();
                        });
                    });
                });
            });
        },
        watch: {
            width: function (newval, oldval) {
                const self = this;
                this.$nextTick(function () {
                    if (oldval === null) {
                        /*
                            If oldval == null is when the width has never been
                            set before. That only occurs when mouting is
                            finished, and onWindowResize has been called and
                            this.width has been changed the first time after it
                            got set to null in the constructor. It is now time
                            to issue layout-ready events as the GridItems have
                            their sizes configured properly.

                            The reason for emitting the layout-ready events on
                            the next tick is to allow for the newly-emitted
                            updateWidth event (above) to have reached the
                            children GridItem-s and had their effect, so we're
                            sure that they have the final size before we emit
                            layout-ready (for this GridLayout) and
                            item-layout-ready (for the GridItem-s).

                            This way any client event handlers can reliably
                            invistigate stable sizes of GridItem-s.
                        */
                        this.$nextTick(() => {
                            this.$emit('layout-ready', self.layout);
                        });
                    }
                    this.updatePlayground();
                });
            },
            layout: function () {
                this.layoutUpdate();
            },
            unit: function () {
                this.eventBus.$emit("setUnit", this.unit);
            },
            isDraggable: function() {
                this.eventBus.$emit("setDraggable", this.isDraggable);
            },
            isResizable: function() {
                this.eventBus.$emit("setResizable", this.isResizable);
            },
            maxRows: function() {
                this.eventBus.$emit("setMaxRows", this.maxRows);
            },
            maxCols: function() {
                this.eventBus.$emit("setMaxCols", this.maxCols);
            }
        },
        methods: {
            layoutUpdate() {
                if (this.layout !== undefined && this.originalLayout !== null) {
                    if (this.layout.length !== this.originalLayout.length) {

                        let diff = this.findDifference(this.layout, this.originalLayout);
                        if (diff.length > 0){
                            if (this.layout.length > this.originalLayout.length) {
                                this.originalLayout = this.originalLayout.concat(diff);
                            } else {
                                this.originalLayout = this.originalLayout.filter(obj => {
                                    return !diff.some(obj2 => {
                                        return obj.i === obj2.i;
                                    });
                                });
                            }
                        }

                        this.lastLayoutLength = this.layout.length;
                    }

                    compact(this.layout, this.verticalCompact);
                    this.updatePlayground();
                    handleNegative(this.layout)
                    this.$emit('layout-updated', this.layout)
                }
            },
            updatePlayground: function () {
                this.mergedStyle = {
                    height: this.containerHeight(),
                    width: this.containerWidth()
                };
            },
            onWindowResize: function () {
                if (this.$refs !== null && this.$refs.item !== null && this.$refs.item !== undefined) {
                    this.width = this.$refs.item.offsetWidth;
                }
                this.eventBus.$emit("resizeEvent");
            },
            containerWidth: function () {
                if (!this.autoSize) return;
                const containerWidth = handleleft(this.layout) * this.unit + handleright(this.layout) * this.unit + 'px';
                return containerWidth;
            },
            containerHeight: function () {
                if (!this.autoSize) return;
                const containerHeight = handletop(this.layout) * this.unit + handlebottom(this.layout) * this.unit + 'px';
                return containerHeight;
            },
            dragEvent: function (eventName, id, x, y, h, w) {
                let l = getLayoutItem(this.layout, id);
                //GetLayoutItem sometimes returns null object
                if (l === undefined || l === null){
                    l = {x:0, y:0}
                }

                if (eventName === "dragmove" || eventName === "dragstart") {
                    this.placeholder.i = id;
                    this.placeholder.x = l.x;
                    this.placeholder.y = l.y;
                    this.placeholder.w = w;
                    this.placeholder.h = h;
                    this.$nextTick(function() {
                        this.isDragging = true;
                    });
                } else {
                    this.$nextTick(function() {
                        this.isDragging = false;
                    });
                }

                // Move the element to the dragged location.
                this.layout = moveElement(this.layout, l, x, y, true, this.preventCollision);
                compact(this.layout, this.verticalCompact);
                // needed because vue can't detect changes on array element properties
                this.eventBus.$emit("compact");
                this.updatePlayground();
                if (eventName === 'dragend') {
                    handleNegative(this.layout)
                    this.$emit('layout-updated', this.layout);
                }
            },
            resizeEvent: function (eventName, id, x, y, h, w) {
                let l = getLayoutItem(this.layout, id);
                //GetLayoutItem sometimes return null object
                if (l === undefined || l === null){
                    l = {h:0, w:0}
                }

                let hasCollisions;
                if (this.preventCollision) {
                    const collisions = getAllCollisions(this.layout, { ...l, w, h }).filter(
                        layoutItem => layoutItem.i !== l.i
                    );
                    hasCollisions = collisions.length > 0;

                    // If we're colliding, we need adjust the placeholder.
                    if (hasCollisions) {
                        // adjust w && h to maximum allowed space
                        let leastX = Infinity,
                        leastY = Infinity;
                        collisions.forEach(layoutItem => {
                        if (layoutItem.x > l.x) leastX = Math.min(leastX, layoutItem.x);
                        if (layoutItem.y > l.y) leastY = Math.min(leastY, layoutItem.y);
                        });

                        if (Number.isFinite(leastX)) l.w = leastX - l.x;
                        if (Number.isFinite(leastY)) l.h = leastY - l.y;
                    }
                }

                if (!hasCollisions) {
                    // Set new width and height.
                    l.w = w;
                    l.h = h;
                }

                if (eventName === "resizestart" || eventName === "resizemove") {
                    this.placeholder.i = id;
                    this.placeholder.x = x;
                    this.placeholder.y = y;
                    this.placeholder.w = l.w;
                    this.placeholder.h = l.h;
                    this.$nextTick(function() {
                        this.isDragging = true;
                    });

                } else {
                    this.$nextTick(function() {
                        this.isDragging = false;
                    });
                }

                compact(this.layout, this.verticalCompact);
                this.eventBus.$emit("compact");
                this.updatePlayground();

                if (eventName === 'resizeend') {
                    handleNegative(this.layout)
                    this.$emit('layout-updated', this.layout);
                }
            },

            // find difference in layouts
            findDifference(layout, originalLayout){

                //Find values that are in result1 but not in result2
                let uniqueResultOne = layout.filter(function(obj) {
                    return !originalLayout.some(function(obj2) {
                        return obj.i === obj2.i;
                    });
                });

                //Find values that are in result2 but not in result1
                let uniqueResultTwo = originalLayout.filter(function(obj) {
                    return !layout.some(function(obj2) {
                        return obj.i === obj2.i;
                    });
                });

                //Combine the two arrays of unique entries#
                return uniqueResultOne.concat(uniqueResultTwo);
            }
        },
    }
</script>
