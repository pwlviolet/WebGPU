<template>
  <canvas id="canvas"></canvas>
</template>
<script setup lang="ts">
import { onMounted } from 'vue';
import triangleVert from './shaders/triangle.vert.wgsl?raw'
import redFrag from './shaders/red.frag.wgsl?raw'
//判断浏览器是否支持webgpu
if (!navigator.gpu)
  throw new Error('no support WebGPU')
onMounted(() => {
run()


})

async function initWebGPU(canvas: HTMLCanvasElement) {
  //创建一个适配器
  const adapter = await navigator.gpu.requestAdapter({
    powerPreference: 'high-performance'
  })
  if (!adapter)
    throw new Error('no adapter')
  //创建一个设备
  const device = await adapter.requestDevice()
  canvas.width = canvas.clientWidth * window.devicePixelRatio
  canvas.height = canvas.clientHeight * window.devicePixelRatio
  if (!device)
    throw new Error('no device')
  //获取上下文
  const context = canvas.getContext('webgpu')
  if (!context)
    throw new Error('no context')
  //获取颜色格式
  // const format = navigator.gpu.getPreferredCanvasFormat ? navigator.gpu.getPreferredCanvasFormat() : context.getPreferredFormat(adapter)
  const format = navigator.gpu.getPreferredCanvasFormat()
  let GPUCanvasConfiguration = {
    device: device,
    format: format
    // usage?: GPUTextureUsageFlags;
    // viewFormats?: Iterable<GPUTextureFormat>;
    // colorSpace?: PredefinedColorSpace;
    // alphaMode?: GPUCanvasAlphaMode;
  }
  context.configure(GPUCanvasConfiguration)
  return{device,adapter,format,context}
}

async function initPipeline(device: GPUDevice, format: GPUTextureFormat): Promise<GPURenderPipeline> {
  const vertex = device.createShaderModule({
    code: triangleVert
  })
  const fragment = device.createShaderModule({
    code: redFrag
  })
  const descriptor: GPURenderPipelineDescriptor = {
    layout: 'auto',
    vertex: {
      module: vertex,
      entryPoint: 'main'
    },
    fragment: {
      module: fragment,
      entryPoint: 'main',
      targets: [{
        format: format
      }]
    },
    primitive: {
      topology: 'triangle-strip'
    }
  }
  //创建渲染管线
  const Pipeline = device.createRenderPipeline(descriptor)
  return Pipeline
}

async function draw(device: GPUDevice, context: GPUCanvasContext, Pipeline: GPURenderPipeline) {
  //创建编码器
  const encoder = device.createCommandEncoder();
  const view = context.getCurrentTexture().createView();
  const renderPassDescriptor: GPURenderPassDescriptor = {
    colorAttachments: [
      {
        view: view,
        clearValue: { r: 0, g: 0, b: 0, a: 1.0 },
        loadOp: 'clear', // clear/load
        storeOp: 'store' // store/discard
      }
    ]
  }
  //创建通道
  const passencoder = encoder.beginRenderPass(renderPassDescriptor);
  //传入渲染管线
  passencoder.setPipeline(Pipeline);
  passencoder.draw(3)
  passencoder.end()
  // webgpu run in a separate process, all the commands will be executed after submit
  device.queue.submit([encoder.finish()])
}

async function run() {
  const canvas=document.querySelector('canvas')
  if(!canvas)
  throw new Error('no canvas')
  const {adapter,device,context,format}=await initWebGPU(canvas)
  const pipeline=await initPipeline(device,format)
  draw(device,context,pipeline)
  
}
</script>






<style >
* {
  margin: 0;
  padding: 0;
  overflow: hidden;
}
#canvas{
  width: 100vw;
  height: 100vh;
}
</style>
