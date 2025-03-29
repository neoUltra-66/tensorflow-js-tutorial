<template>
  <div>
    <canvas
      ref="canvas"
      width="500"
      height="300"
      @mousedown="startDrawing"
      @mouseup="stopDrawing"
      @mousemove="draw"
      style="border: 1px solid #000; cursor: crosshair;"
    ></canvas>
    
    <button @click="clearCanvas" style="margin-top: 10px;">清除画布</button>
    
    <button @click="predictDigit" style="margin-top: 10px;">预测数字</button>
    <p v-if="prediction !== null">预测结果: {{ prediction }}</p>
  </div>
</template>

<script setup>
import { ref, onMounted, reactive } from "vue";
import * as tf from "@tensorflow/tfjs";
    const canvas = ref(null);
    const ctx = ref(null);
    let model = reactive(null)
    const isDrawing = ref(false);
    const pos = ref({ x: 0, y: 0 });
    const prediction = ref(null);

    const setPosition = (event) => {
      const rect = canvas.value.getBoundingClientRect();
      pos.value.x = event.clientX - rect.left;
      pos.value.y = event.clientY - rect.top;
    };

    const startDrawing = (event) => {
      isDrawing.value = true;
      setPosition(event);
    };

    const stopDrawing = () => {
      isDrawing.value = false;
      ctx.value.beginPath();
    };

    const draw = (event) => {
      if (!isDrawing.value) return;

      ctx.value.beginPath();
      ctx.value.lineWidth = 30;
      ctx.value.lineCap = "round";
      ctx.value.strokeStyle = "#000";

      ctx.value.moveTo(pos.value.x, pos.value.y);
      setPosition(event);
      ctx.value.lineTo(pos.value.x, pos.value.y);

      ctx.value.stroke();
    };

    const predictDigit = () => {
      if (!model) {
        alert("模型尚未加载完成");
        return;
      }
      const imageData = ctx.value.getImageData(0, 0, canvas.value.width, canvas.value.height);
      let img = tf.browser.fromPixels(imageData);
      
      // 调整为 28x28 尺寸并转为灰度
      img = tf.image.resizeBilinear(img, [28, 28]).toFloat();

      img = img.mean(2)
        .toFloat()
        .expandDims(0)
        .expandDims(-1);

      img = img.div(tf.scalar(255.0));

      img = tf.scalar(1.0).sub(img);

      // 做出预测
      const result = model.predict(img).arraySync();

      prediction.value = result[0].indexOf(Math.max(...result[0]));
    };

    const clearCanvas = () => {
      ctx.value.clearRect(0, 0, canvas.value.width, canvas.value.height);
      ctx.value.fillStyle = "#FFFFFF";
      ctx.value.fillRect(0, 0, canvas.value.width, canvas.value.height);
    };

    onMounted(async () => {
      const canvasElement = canvas.value;
      ctx.value = canvasElement.getContext("2d");
      ctx.value.fillStyle = "#FFFFFF";
      ctx.value.fillRect(0, 0, canvasElement.width, canvasElement.height);
      try {
        model = await tf.loadLayersModel("/src/Model_CNN/model.json");
        console.log("模型加载完成")
      } catch (error) {
        console.error("模型加载失败:", error);
      }
      
    });

</script>

<style scoped>
button {
  font-size: 14px;
  background-color: #4CAF50;
  color: white;
  border: none;
  padding: 10px 20px;
  text-align: center;
  cursor: pointer;
  border-radius: 5px;
}

button:hover {
  background-color: #45a049;
}
</style>
