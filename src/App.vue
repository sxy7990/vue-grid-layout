<template>
    <div id="app">
        <h1 style="text-align: center">Vue Grid Layout</h1>
        <div>
            <div class="layoutJSON">
                Displayed as <code>[x, y, w, h]</code>:
                <div class="columns">
                    <div class="layoutItem" v-for="item in layout" :key="item.i">
                        <b>{{item.i}}</b>: [{{item.x}}, {{item.y}}, {{item.w}}, {{item.h}}]
                    </div>
                </div>
            </div>
        </div>
        <div id="content">
            <button @click="decreaseWidth">Decrease Width</button>
            <button @click="increaseWidth">Increase Width</button>
            <button @click="addItem">Add an item</button>
            <button @click="addItemDynamically">Add an item dynamically</button>
            <input type="checkbox" v-model="draggable"/> Draggable
            <input type="checkbox" v-model="resizable"/> Resizable
            <input type="checkbox" v-model="preventCollision"/> Prevent Collision
            <div style="margin-top: 10px;margin-bottom: 10px;">
                Row Height: <input type="number" v-model="rowHeight"/> Col nums: <input type="number" v-model="colNum"/>
                Margin x: <input type="number" v-model="marginX"/> Margin y: <input type="number" v-model="marginY"/>
            </div>
            <div style="width: 1000px; height: 720px; overflow: auto; background-color: rgba(23, 23, 112, 0.1);">
                <grid-layout
                    :margin="[parseInt(marginX), parseInt(marginY)]"
                        :layout.sync="layout"
                        :col-num="parseInt(colNum)"
                        :col-width="colWidth"
                        :row-height="rowHeight"
                        :is-draggable="draggable"
                        :is-resizable="resizable"
                        :prevent-collision="preventCollision"
                        :vertical-compact="compact"
                        :use-css-transforms="true"
                        @layout-created="layoutCreatedEvent"
                        @layout-before-mount="layoutBeforeMountEvent"
                        @layout-mounted="layoutMountedEvent"
                        @layout-ready="layoutReadyEvent"
                        @layout-updated="layoutUpdatedEvent"
                        @breakpoint-changed="breakpointChangedEvent"
                >
                    <grid-item v-for="item in layout" :key="item.i"
                            :static="item.static"
                            :x="item.x"
                            :y="item.y"
                            :w="item.w"
                            :h="item.h"
                            :i="item.i"
                            :min-w="item.minW"
                            :max-w="item.maxW"
                            :min-x="item.minX"
                            :max-x="item.maxX"
                            :min-y="item.minY"
                            :max-y="item.maxY"
                            @resize="resize"
                            @move="move"
                            @resized="resized"
                            @container-resized="containerResized"
                            @moved="moved"
                    >
                        <test-element :text="item.i" @removeItem="removeItem($event)"></test-element>
                    </grid-item>
                </grid-layout>
            </div>
            <hr/>
        </div>
    </div>
</template>

<script>
    import GridItem from './components/GridItem.vue';
    import GridLayout from './components/GridLayout.vue';
    import TestElement from './components/TestElement.vue';
    import CustomDragElement from './components/CustomDragElement.vue';

    let testLayout = [
        {"x":0,"y":0,"w":2,"h":2,"i":"0", resizable: true, draggable: true, static: false},
        {"x":2,"y":0,"w":2,"h":4,"i":"1", resizable: null, draggable: null, static: true},
        {"x":4,"y":0,"w":2,"h":5,"i":"2", resizable: false, draggable: false, static: false},
        {"x":6,"y":0,"w":2,"h":3,"i":"3", resizable: false, draggable: false, static: false},
        {"x":8,"y":0,"w":2,"h":3,"i":"4", resizable: false, draggable: false, static: false},
        {"x":10,"y":0,"w":2,"h":3,"i":"5", resizable: false, draggable: false, static: false},
        {"x":0,"y":5,"w":2,"h":5,"i":"6", resizable: false, draggable: false, static: false},
        {"x":2,"y":5,"w":2,"h":5,"i":"7", resizable: false, draggable: false, static: false},
    ];

    export default {
        name: 'app',
        components: {
            GridLayout,
            GridItem,
            TestElement,
            CustomDragElement,
        },
        data () {
            return {
                layout: JSON.parse(JSON.stringify(testLayout)),
                layout2: JSON.parse(JSON.stringify(testLayout)),
                draggable: true,
                resizable: true,
                preventCollision: true,
                compact: true,
                rowHeight: 40,
                colWidth: 40,
                colNum: 12,
                index: 0,
                marginX: 10,
                marginY: 10,
            }
        },
        mounted: function () {
            this.index = this.layout.length;
        },
        methods: {
            clicked: function() {
                window.alert("CLICK!");
            },
            increaseWidth: function() {
                let width = document.getElementById("content").offsetWidth;
                width += 20;
                document.getElementById("content").style.width = width+"px";
            },
            decreaseWidth: function() {
                let width = document.getElementById("content").offsetWidth;
                width -= 20;
                document.getElementById("content").style.width = width+"px";
            },
            removeItem: function(i) {
                console.log("### REMOVE " + i);
                const index = this.layout.map(item => item.i).indexOf(i);
                this.layout.splice(index, 1);
            },
            addItem: function() {
                let item = {"x":0,"y":0,"w":2,"h":2,"i":this.index+"", whatever: "bbb"};
                this.index++;
                this.layout.push(item);
            },
            addItemDynamically: function() {
                const x = (this.layout.length * 2) % (this.colNum || 12);
                const y = this.layout.length + (this.colNum || 12);
                console.log("X=" + x + " Y=" + y)
                let item = {
                  x: x,
                  y: y,
                  w: 2,
                  h: 2,
                  i: this.index+"",
                }
                this.index++;
                this.layout.push(item);
            },
            move: function(i, newX, newY){
                // console.log("MOVE i=" + i + ", X=" + newX + ", Y=" + newY);
            },
            resize: function(i, newH, newW, newHPx, newWPx){
                // console.log("RESIZE i=" + i + ", H=" + newH + ", W=" + newW + ", H(px)=" + newHPx + ", W(px)=" + newWPx);
            },
            moved: function(i, newX, newY){
                // console.log("### MOVED i=" + i + ", X=" + newX + ", Y=" + newY);
            },
            resized: function(i, newH, newW, newHPx, newWPx){
                // console.log("### RESIZED i=" + i + ", H=" + newH + ", W=" + newW + ", H(px)=" + newHPx + ", W(px)=" + newWPx);
            },
            containerResized: function(i, newH, newW, newHPx, newWPx){
                // console.log("### CONTAINER RESIZED i=" + i + ", H=" + newH + ", W=" + newW + ", H(px)=" + newHPx + ", W(px)=" + newWPx);
            },

            layoutCreatedEvent: function(newLayout){
                // console.log("Created layout: ", newLayout)
            },
            layoutBeforeMountEvent: function(newLayout){
                // console.log("beforeMount layout: ", newLayout)
            },
            layoutMountedEvent: function(newLayout){
                // console.log("Mounted layout: ", newLayout)
            },
            layoutReadyEvent: function(newLayout){
                // console.log("Ready layout: ", newLayout)
            },
            layoutUpdatedEvent: function(newLayout){
                // console.log("Updated layout: ", newLayout)
            },
            breakpointChangedEvent: function(newBreakpoint, newLayout){
                // console.log("breakpoint changed breakpoint=", newBreakpoint, ", layout: ", newLayout );
            }

        },
    }
</script>

<style lang="scss">
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
}
</style>
