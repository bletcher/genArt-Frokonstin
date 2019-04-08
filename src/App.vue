<template>
  <div>
    <v-stage
      ref="stage"
      :config="configKonva"
      @dragstart="handleDragstart"
      @dragend="handleDragend"
    >
      <v-layer ref="layer">
        <v-line v-for="item in listPolygonScreen" :key="item.id" :config="item"></v-line>
        <v-line v-for="item in listSegment" :key="item.id" :config="item"></v-line>
      </v-layer>
      <v-layer ref="dragLayer"></v-layer>
    </v-stage>
  </div>
</template>

<script>
const width = window.innerWidth;
const height = window.innerHeight;
let vm = {};
export default {
  data() {
    return {
      listPolygon: [],
      listPolygonScreen: [],
      listCon: [],
      listSegment: [],
      configKonva: {
        width: width,
        height: height
      }
    };
  },

  mounted() {
    vm = this;
    const stage = vm.$refs.stage.getStage();
    const initX = stage.width() / 1.5;
    const initY = stage.height() / 4;
    const scale = 200;

    const edges = 7; //# Number of edges of the original polygon
    let nIter = 3; //# Number of iterations
    let pond = 0.5; //# Weight to calculate the point on the middle of each edge
    let step = 2; // # No of times to draw mid-segments before connect ending points
    let angle = Math.PI / 2; //# angle of mid-segment with the edge
    let curve = 5; //# Curvature of curves
    let fun = function(d) {
      return 0.2;
    };

    vm.listPolygonScreen.length = 0;
    console.log("before 1 polygon", vm.listPolygonScreen);

    //first lines

    this.polygon(0, edges, scale, initX, initY);
    for (let n = 1; n < edges; n++) {
      this.polygon(n, edges, scale);
    }
    vm.listPolygonScreen.push.apply(vm.listPolygonScreen, vm.listPolygon);
    console.log("after 1 polygon", 1, vm.listPolygonScreen);

    // add nIter rings
    for (let r = 1; r < nIter; r++) {
      console.log("in r loop", r);
      this.midPoints(0, scale, edges, r, pond, angle, fun);
      //this.midPoints(1, scale, edges, 2);
      for (let n = 1; n < edges; n++) {
        this.midPoints(n, scale, edges, r, pond, angle, fun);
        this.conPoints(n, edges, r, curve);
      }
      // // last line
      this.conPoints(edges, edges, r, curve);

      vm.listPolygonScreen.push.apply(vm.listPolygonScreen, vm.listCon);
      vm.listCon.length = 0;
    }
    console.log("hold", this.listPolygonScreen);

    // console.log("after loop polygon", vm.listPolygon);
    // console.log("after loop seg", vm.listSegment);
    // console.log("after loop con", vm.listCon);
    console.log("map i", this.listPolygonScreen.map(tmp => tmp.i));
    console.log("map ringNum", this.listPolygonScreen.map(tmp => tmp.ringNum));
    // officers.map(officer => officer.id)
  },

  methods: {
    polygon(n, edges, scale, initX, initY) {
      const xStart = n == 0 ? initX : vm.listPolygon[n - 1].xEnd;
      const yStart = n == 0 ? initY : vm.listPolygon[n - 1].yEnd;
      const xOffset = Math.cos(((n + 1) * 2 * Math.PI) / edges) * scale;
      const yOffset = Math.sin(((n + 1) * 2 * Math.PI) / edges) * scale;

      vm.listPolygon.push({
        x: xStart,
        y: yStart,
        xEnd: xStart + xOffset,
        yEnd: yStart + yOffset,
        points: [0, 0, xOffset, yOffset],
        ringNum: 0,
        i: n,
        stroke: "red",
        source: "polygon"
      });
    },
    midPoints(n, scale, edges, ringNum, pond, angle, fun) {
      const i = n + edges * (ringNum - 1);
      console.log("fun", i, fun(i), ringNum, fun(ringNum));
      var xEndSegmentOffset = null;
      var yEndSegmentOffset = null;
      const ang =
        Math.atan2(
          vm.listPolygonScreen[i].yEnd - vm.listPolygonScreen[i].y,
          vm.listPolygonScreen[i].xEnd - vm.listPolygonScreen[i].x
        ) + angle;

      if (ringNum > 1) {
        xEndSegmentOffset = -fun(ringNum) * Math.cos(ang) * scale; // not sure why this is necessary
        yEndSegmentOffset = -fun(ringNum) * Math.sin(ang) * scale;
      } else {
        xEndSegmentOffset = fun(ringNum) * Math.cos(ang) * scale;
        yEndSegmentOffset = fun(ringNum) * Math.sin(ang) * scale;
      }
      const xSeg =
        pond * vm.listPolygonScreen[i].x +
        (1 - pond) * vm.listPolygonScreen[i].xEnd;
      const ySeg =
        pond * vm.listPolygonScreen[i].y +
        (1 - pond) * vm.listPolygonScreen[i].yEnd;

      vm.listSegment.push({
        angle: angle,
        x: xSeg,
        y: ySeg,
        xEndSegmentOffset: xEndSegmentOffset,
        yEndSegmentOffset: yEndSegmentOffset,
        xEnd: xSeg + xEndSegmentOffset,
        yEnd: ySeg + yEndSegmentOffset,
        points: [0, 0, xEndSegmentOffset, yEndSegmentOffset],
        ringNum: ringNum,
        i: i,
        stroke: "green",
        source: "midPoints"
      });
    },
    conPoints(n, edges, ringNum, curve) {
      const i = n + edges * (ringNum - 1);
      var xSegOffset = null;
      var ySegOffset = null;

      if (n == edges) {
        xSegOffset = -(
          vm.listSegment[edges * (ringNum - 1)].xEnd -
          vm.listSegment[i - 1].xEnd
        );
        ySegOffset = -(
          vm.listSegment[edges * (ringNum - 1)].yEnd -
          vm.listSegment[i - 1].yEnd
        );
        vm.listCon.push({
          x: vm.listSegment[edges * (ringNum - 1)].xEnd,
          y: vm.listSegment[edges * (ringNum - 1)].yEnd,
          points: [0, 0, xSegOffset, ySegOffset],
          xEnd: vm.listSegment[edges * (ringNum - 1)].xEnd + xSegOffset,
          yEnd: vm.listSegment[edges * (ringNum - 1)].yEnd + ySegOffset,
          tension: curve,
          ringNum: ringNum,
          i: i,
          stroke: "blue",
          source: "conPoints2"
        });
      } else {
        xSegOffset = -(vm.listSegment[i].xEnd - vm.listSegment[i - 1].xEnd);
        ySegOffset = -(vm.listSegment[i].yEnd - vm.listSegment[i - 1].yEnd);

        vm.listCon.push({
          x: vm.listSegment[i].xEnd,
          y: vm.listSegment[i].yEnd,
          points: [0, 0, xSegOffset, ySegOffset],
          xEnd: vm.listSegment[i].xEnd + xSegOffset,
          yEnd: vm.listSegment[i].yEnd + ySegOffset,
          tension: curve,
          ringNum: ringNum,
          i: i,
          stroke: "blue",
          source: "conPoints1"
        });
      }
    },
    handleDragstart(e) {
      const shape = e.target;
      const dragLayer = vm.$refs.dragLayer.getNode();
      const stage = vm.$refs.stage.getNode();
      // moving to another layer will improve dragging performance
      shape.moveTo(dragLayer);
      stage.draw();
      shape.setAttrs({
        shadowOffsetX: 15,
        shadowOffsetY: 15,
        scaleX: shape.getAttr("startScale") * 1.2,
        scaleY: shape.getAttr("startScale") * 1.2
      });
    },
    handleDragend(e) {
      const shape = e.target;
      const layer = vm.$refs.layer.getNode();
      const stage = vm.$refs.stage.getNode();
      shape.moveTo(layer);
      stage.draw();
      shape.to({
        duration: 0.5,
        easing: Konva.Easings.ElasticEaseOut,
        scaleX: shape.getAttr("startScale"),
        scaleY: shape.getAttr("startScale"),
        shadowOffsetX: 5,
        shadowOffsetY: 5
      });
    }
  }
};
</script>