<template>
  <div class="page">
    <div class="draw-board">
      <div class="tool">
        <div class="tool-left">
          <div :class="['tool-item', drawType === '' ? 'active' : undefined]">
            <el-tooltip effect="dark" content="选择" placement="top">
              <svg-icon iconname="xuanze" @click="chooseObject"></svg-icon>
            </el-tooltip>
          </div>
          <div :class="['tool-item', drawType === 'importBG' ? 'active' : undefined]">
            <el-tooltip effect="dark" content="导入背景" placement="top">
              <el-upload ref="upload" :limit="1" class="photo-upload" :on-change="uploadChange" :auto-upload="false"
                ><span class="label">upload</span></el-upload
              >
            </el-tooltip>
            <svg-icon iconname="tupian"></svg-icon>
          </div>
          <div :class="['tool-item', drawType === 'drawPloygon' ? 'active' : undefined]">
            <el-tooltip effect="dark" content="画多边形" placement="top">
              <svg-icon iconname="duobianxing" @click="drawPloygon"></svg-icon>
            </el-tooltip>
          </div>
          <div :class="['tool-item', drawType === 'drawText' ? 'active' : undefined]">
            <el-tooltip effect="dark" content="添加文字" placement="top">
              <svg-icon iconname="wenzi" @click="drawText"></svg-icon>
            </el-tooltip>
          </div>
          <div :class="['tool-item', drawType === 'freeDraw' ? 'active' : undefined]">
            <el-tooltip effect="dark" content="自由绘画" placement="top">
              <svg-icon iconname="huabi" @click="freeDraw"></svg-icon>
            </el-tooltip>
          </div>
          <div :class="['tool-item', originStackRef.length === 0 ? 'disabled' : undefined]">
            <el-tooltip effect="dark" content="撤销" placement="top">
              <svg-icon iconname="shangyibu" @click="prevDraw"></svg-icon>
            </el-tooltip>
          </div>
          <div :class="['tool-item', recoverStackRef.length === 0 ? 'disabled' : undefined]">
            <el-tooltip effect="dark" content="恢复" placement="top">
              <svg-icon iconname="xiayibu" @click="nextDraw"></svg-icon>
            </el-tooltip>
          </div>
          <div class="tool-item">
            <el-tooltip effect="dark" content="删除" placement="top">
              <svg-icon iconname="shanchu" @click="deleteObject"></svg-icon>
            </el-tooltip>
          </div>
          <div class="tool-item">
            <el-tooltip effect="dark" content="导出为图片" placement="top">
              <svg-icon iconname="baocun" @click="save"></svg-icon>
            </el-tooltip>
          </div>
        </div>
        <div class="tool-right">
          <div class="color-tool">
            <el-color-picker v-model="color" show-alpha :predefine="predefineColors" />
          </div>
          <div class="border-tool">
            <el-dropdown trigger="click" class="border-menu">
              <div class="menu-label">
                <div :class="['border show', `border_${borderWidth}`]"></div>
                <el-icon>
                  <arrow-down />
                </el-icon>
              </div>
              <template #dropdown>
                <el-dropdown-menu class="border-menu-list">
                  <el-dropdown-item v-for="item in borders" :key="item">
                    <div :class="['border', `border_${item}`]" @click="changeBorder(item)"></div>
                  </el-dropdown-item>
                </el-dropdown-menu>
              </template>
            </el-dropdown>
          </div>
        </div>
      </div>
      <canvas id="cas" width="1200" height="654"></canvas>
    </div>
  </div>
</template>

<script setup>
import { onMounted, reactive, ref, watch } from "vue";
import SvgIcon from "./components/SvgIcon.vue";
import { fabric } from "fabric";
import { ArrowDown } from "@element-plus/icons-vue";
const currIndex = ref(-1);
// 已经绘制的对象栈
let originStackRef = ref([]);
let recoverStackRef = ref([]);
let photoUrl = ref("");
let originStack = [];
let recoverStack = [];
onMounted(() => {
  initCas();
});
// 画布对象
let canvas = {};
// 绘制对象
let polygon_line = {};
let polygon_polyline = {};
let polygon = {};
let textbox = {};
// 选中的对象
let selectObject = {
  list: []
};
// 当前绘制对象坐标
let points = [];
// 绘制类型
let drawType = ref("");
// 绘制颜色
let color = ref("rgba(0, 115, 230,1)");
// 绘制粗细
let borderWidth = ref(1);
// 预选颜色
let predefineColors = ref(["#ff4500", "#ff8c00", "#ffd700", "#90ee90", "#00ced1", "#1e90ff", "#c71585"]);
// 预选粗细
let borders = ref([1, 2, 4, 8]);
// 监听绘制类型变化
watch(drawType, newState => {
  if (newState === "freeDraw") {
    canvas.isDrawingMode = true;
  } else {
    canvas.isDrawingMode = false;
  }
});
// 监听粗细变化
watch(borderWidth, newState => {
  canvas.freeDrawingBrush.width = newState;
});
// 监听颜色变化
watch(color, newState => {
  canvas.freeDrawingBrush.color = newState;
});
const initCas = () => {
  canvas = new fabric.Canvas("cas");
  // 初始化画布状态
  canvas.hasDraw = false;
  canvas.zoom = 1;
  // 注册画布事件
  initEvent();
};
const initEvent = () => {
  canvas.on("mouse:down", onMouseDown);
  canvas.on("mouse:move", onMouseMove);
  canvas.on("mouse:up", onMouseUp);
  canvas.on("mouse:wheel", onMouseWheel);
  canvas.on("mouse:dblclick", onMouseDBClick);
  canvas.on("selection:created", onObjectSelect);
  canvas.on("selection:cleared", onObjectCancelSelect);
};
// 选中事件
const onObjectSelect = opt => {
  selectObject.list = opt.selected;
};
// 取消选中
const onObjectCancelSelect = opt => {
  selectObject.list = [];
};
const onMouseDown = opt => {
  let evt = opt.e;
  if (evt.altKey === true) {
    canvas.isDragging = true;
    canvas.lastPosX = evt.offsetX;
    canvas.lastPosY = evt.offsetY;
  }
};
const onMouseMove = opt => {
  if (canvas.isDragging) {
    let evt = opt.e;
    let vpt = canvas.viewportTransform;
    vpt[4] += evt.offsetX - canvas.lastPosX;
    vpt[5] += evt.offsetY - canvas.lastPosY;
    canvas.requestRenderAll();
    canvas.lastPosX = evt.offsetX;
    canvas.lastPosY = evt.offsetY;
  }
  if (canvas.isDragging) return;
  if (canvas.hasDraw) return;
  let point = resizePos(opt);
  drawpath(point);
};
const onMouseUp = opt => {
  canvas.setViewportTransform(canvas.viewportTransform);
  canvas.isDragging = false;
  if (opt.currentTarget && drawType.value !== "drawPloygon") {
    // 推入画笔绘制对象
    originStackRef.value.push(opt.currentTarget);
    originStack.push(opt.currentTarget);
  }
};
const onMouseWheel = opt => {
  let delta = opt.e.deltaY;
  let zoom = canvas.getZoom();
  zoom *= 0.999 ** delta;
  if (zoom > 20) zoom = 20;
  if (zoom < 0.01) zoom = 0.01;
  canvas.setZoom(zoom);
  canvas.zoom = zoom;
};
const onMouseDBClick = opt => {
  if (canvas.hasDraw) return;
  if (points.length >= 3) {
    canvas.remove(polygon_line);
    canvas.remove(polygon_polyline);
    polygon = new fabric.Polygon(points, {
      fill: "rgba(0, 115, 230,0.2)",
      stroke: color.value,
      strokeWidth: 5
    });
    canvas.add(polygon);
    originStackRef.value.push(polygon);
    originStack.push(polygon);
    if (drawType.value === "drawPloygon") {
      drawType.value = "";
      canvas.hasDraw = true;
    }
  }
};
const drawPloygon = () => {
  canvas.hasDraw = false;
  points = [];
  drawType.value = "drawPloygon";
  canvas.on("mouse:down", opt => {
    if (canvas.isDragging) return;
    if (canvas.hasDraw) return;
    let point = resizePos(opt);
    points.push({
      x: point.x,
      y: point.y
    });
    drawpath();
  });
};
const drawpath = point => {
  let len = points.length;
  if (len === 0) return;
  canvas.remove(polygon_line);
  canvas.remove(polygon_polyline);
  if (point) {
    let start = {
      x: points[points.length - 1].x,
      y: points[points.length - 1].y
    };
    let end = {
      x: point.x,
      y: point.y
    };
    polygon_line = new fabric.Line([start.x, start.y, end.x, end.y], {
      stroke: color.value,
      strokeWidth: 5
    });
    canvas.add(polygon_line);
  }
  polygon_polyline = new fabric.Polyline(points, {
    fill: "rgba(0, 115, 230,0.2)",
    stroke: color.value,
    strokeWidth: 5
  });
  canvas.add(polygon_polyline);
};
const uploadChange = uploadFile => {
  photoUrl.value = URL.createObjectURL(uploadFile.raw);
  importPhoto(photoUrl.value);
};
const importPhoto = url => {
  fabric.Image.fromURL(url, img => {
    canvas.setBackgroundImage(img, canvas.renderAll.bind(canvas), {
      scaleX: canvas.width / img.width,
      scaleY: canvas.height / img.height
    });
    drawType.value = "importBG";
  });
};
const drawText = () => {
  drawType.value = "drawText";
  canvas.on("mouse:down", opt => {
    if (drawType.value !== "drawText") return;
    let point = resizePos(opt);
    textbox = new fabric.Textbox("输入文字", {
      width: 250,
      top: point.y,
      left: point.x,
      fill: color.value
    });
    canvas.add(textbox);
    originStackRef.value.push(textbox);
    originStack.push(textbox);
    canvas.hasDraw = true;
    drawType.value = "";
  });
};
const freeDraw = () => {
  drawType.value = "freeDraw";
  canvas.freeDrawingBrush.color = color.value;
  canvas.freeDrawingBrush.width = borderWidth.value;
};
const deleteObject = () => {
  selectObject.list.forEach(item => {
    canvas.remove(item);
  });
};
const resizePos = opt => {
  let vpt = canvas.viewportTransform;
  let spaceX = -vpt[4];
  let spaceY = -vpt[5];
  let offsetX = opt.e.offsetX + spaceX;
  let offsetY = opt.e.offsetY + spaceY;
  if (canvas.zoom >= 1) {
    offsetX = offsetX / canvas.zoom;
    offsetY = offsetY / canvas.zoom;
  } else {
    offsetX = offsetX * (1 / canvas.zoom);
    offsetY = offsetY * (1 / canvas.zoom);
  }
  let point = {
    x: offsetX,
    y: offsetY
  };
  return point;
};
const chooseObject = () => {
  drawType.value = "";
};
const changeBorder = border => {
  console.log(border);
  borderWidth.value = border;
};
const save = () => {
  // 恢复缩放
  canvas.setZoom(1);
  // 恢复平移
  let vpt = canvas.viewportTransform;
  vpt[4] = 0;
  vpt[5] = 0;
  canvas.requestRenderAll();
  let photoUrl = canvas.toDataURL({
    format: "jpg",
    quality: 1
  });
  let a = document.createElement("a");
  let event = new MouseEvent("click");
  a.download = "photo";
  document.body.append(a);
  a.href = photoUrl;
  a.dispatchEvent(event);
};
const prevDraw = () => {
  let item = originStack.pop();
  originStackRef.value.pop();
  if (item) {
    canvas.remove(item);
    recoverStack.unshift(item);
    recoverStackRef.value.unshift(item);
  }
};
const nextDraw = () => {
  let item = recoverStack.shift();
  recoverStackRef.value.shift();
  if (item) {
    canvas.add(item);
    originStack.push(item);
    originStackRef.value.push(item);
  }
};
</script>
<style scoped>
.page {
  width: 100vw;
  height: 100vh;
  background: #333;
  overflow: hidden;
}
.draw-board {
  margin-top: 100px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}
.draw-board .tool {
  width: 1200px;
  height: 50px;
  padding: 0 30px;
  background: #fff;
  border-radius: 5px;
  margin-bottom: 20px;
  display: flex;
  justify-content: space-between;
}
.tool-left,
.tool-right {
  height: 100%;
  display: flex;
  align-items: center;
}

.draw-board #cas {
  border: 1px solid #eee;
  background: #fff;
  border-radius: 5px;
}
.tool-left .tool-item {
  width: 30px;
  padding: 3px 0;
  text-align: center;
  cursor: pointer;
  border-radius: 3px;
  margin-right: 10px;
  position: relative;
}
.photo-upload {
  position: absolute;
  width: 100%;
  height: 100%;
  opacity: 0;
}
.tool .tool-item:hover {
  color: #1677ff;
}
.tool .tool-item.active {
  background: #1677ff;
  color: #fff;
}
.tool .tool-item.active:hover {
  color: #fff;
}
.tool .tool-item.disabled {
  color: #999;
  cursor: no-drop;
}
.color-tool {
  margin-right: 10px;
}
.border-menu .menu-label {
  display: flex;
  align-items: center;
  border: 1px solid #eee;
  padding: 5px;
  border-radius: 5px;
}
.border {
  width: 30px;
  padding: 2px 0;
  border-bottom: 1px solid #666;
}
.border.show {
  padding: 0;
  margin-right: 5px;
}

.border.border_1 {
  border-bottom: 1px solid #666;
}
.border.border_2 {
  border-bottom: 2px solid #666;
}
.border.border_4 {
  border-bottom: 4px solid #666;
}
.border.border_8 {
  border-bottom: 8px solid #666;
}
</style>
