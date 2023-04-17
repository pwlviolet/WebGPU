<template>
  <canvas id="canvas"></canvas>
</template>
<script setup lang="ts">
import { guardReactiveProps, onMounted } from 'vue';
import positionVert from './shaders/basic.vert.wgsl?raw'
import uniformfrag from './shaders/position.frag.wgsl?raw'
import imageTexture from './shaders/imageTexture.frag.wgsl?raw'
import * as cube from '../util/cube'
import * as math from '../util/math'
import * as dat from 'dat.gui';
//导入图片
import textureUrl from '/texture.webp?url'
//判断浏览器是否支持webgpu
if (!navigator.gpu)
  throw new Error('no support WebGPU')
//全局变量声明
let aspect = window.innerWidth / window.innerHeight;
const position = { x: -2, y: 0, z: -5 }
const scale = { x: 1, y: 1, z: 1 }
const rotation = { x: 0, y: 0, z: 0 }
//第二组
const position2 = { x: 2, y: 0, z: -5 }
const scale2 = { x: 1, y: 1, z: 1 }
const rotation1 = { x: 0, y: 0, z: 0 }
onMounted(() => {
  run()
  AddGUI()
}

)
function AddGUI() {
  //使用dat.gui
  const gui = new dat.GUI();
  const positionFolder = gui.addFolder('位置');
  positionFolder.add(position, 'x')
    .min(-4) //最小x位置
    .max(2)//最大x位置
    .step(0.1)//精度
    .name("x轴")//命名
  positionFolder.add(position, 'y')
    .min(-2) //最小y位置
    .max(2)//最大y位置
    .step(0.1)//精度
    .name("y轴")//命名
  positionFolder.add(position, 'z')
    .min(-10) //最小z位置
    .max(-5)//最大z位置
    .step(0.1)//精度
    .name("z轴")//命名
  const scaleFolder = gui.addFolder('缩放');
  scaleFolder.add(scale, 'x')
    .min(0.1)
    .max(10)
    .step(0.1)
    .name('x')
  scaleFolder.add(scale, 'y')
    .min(0.1)
    .max(10)
    .step(0.1)
    .name('y')
  scaleFolder.add(scale, 'z')
    .min(0.1)
    .max(10)
    .step(0.1)
    .name('z')


}


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
  return { device, adapter, format, context }
}

async function initPipeline(device: GPUDevice, format: GPUTextureFormat) {
  //创建顶点buffer
  const vertexbuffer = device.createBuffer({
    size: cube.vertex.byteLength,
    usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST
  })
  //创建uniformbuffer
  const uniformbuffer = device.createBuffer({
    size: 4 * 4, //4*float32
    usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST
  })
  //创建mvp buffer
  const mvpbuffer = device.createBuffer({
    // size: 4 * 4 * 4,
    size: 256 * 2,
    usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST
  })
  // const mvpbuffer1=device.createBuffer({
  //   size:4*4*4,
  //   usage:GPUBufferUsage.UNIFORM|GPUBufferUsage.COPY_DST
  // })
  //写入device
  device.queue.writeBuffer(vertexbuffer, 0, cube.vertex)
  // device.queue.writeBuffer(uniformbuffer, 0, new Float32Array([1,1,0,1]))

  const vertex = device.createShaderModule({
    code: positionVert
  })
  const fragment = device.createShaderModule({
    code: uniformfrag
  })
  const descriptor: GPURenderPipelineDescriptor = {
    layout: 'auto',
    vertex: {
      module: vertex,
      entryPoint: 'main',
      buffers: [{
        arrayStride: 5 * 4, // 3 position 2 uv,
        attributes: [
          {
            // position
            shaderLocation: 0,
            offset: 0,
            format: 'float32x3',
          },
          {
            // uv
            shaderLocation: 1,
            offset: 3 * 4,
            format: 'float32x2',
          }
        ]
      }]
      // buffers: [{
      //           arrayStride: 5 * 4, // 3 float32,2uv
      //           attributes: [
      //               {
      //                   // position xyz
      //                   shaderLocation: 0,
      //                   offset: 0,
      //                   format: 'float32x3',
      //               }
      //           ]
      //             }]
    },
    // fragment: {
    //   module: fragment,
    //   entryPoint: 'main',
    //   targets: [{
    //     format: format
    //   }]
    // },
    fragment: {
            module: device.createShaderModule({
                code: imageTexture,
            }),
            entryPoint: 'main',
            targets: [
                {
                    format: format
                }
            ]
        },
    primitive: {
      topology: 'triangle-list',
      //去除背面
      cullMode: 'back',
    }
    //深度测试
    // ,      depthStencil: {
    //         depthWriteEnabled: true,
    //         depthCompare: 'less',
    //         format: 'depth24plus',
    //     }
  }

  //创建渲染管线
  const Pipeline = device.createRenderPipeline(descriptor)
  const uniformgroup = device.createBindGroup({
    layout: Pipeline.getBindGroupLayout(0),
    entries:
      [
        //   {
        //   binding:0,
        //   resource:{
        //     buffer:uniformbuffer
        //   }
        // },
        {
          binding: 0,
          resource: {
            buffer: mvpbuffer,
            offset: 0
          }
        }
      ]
  })
  const mvp2group = device.createBindGroup({
    layout: Pipeline.getBindGroupLayout(0),
    entries: [{
      binding: 0,
      resource: {
        buffer: mvpbuffer,
        offset: 256,
      }
    }]
  })
  return { Pipeline, vertexbuffer, uniformgroup, mvpbuffer, mvp2group }
}

async function draw(device: GPUDevice, context: GPUCanvasContext, Pipeline: GPURenderPipeline, vertexbuffer: GPUBuffer, uniformgroup: GPUBindGroup, mvp2group: GPUBindGroup,textureGroup:GPUBindGroup) {
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
  //设置顶点缓冲
  passencoder.setVertexBuffer(0, vertexbuffer)
  //设置纹理
  passencoder.setBindGroup(1, textureGroup)
  passencoder.setBindGroup(0, uniformgroup)
  passencoder.draw(cube.vertexCount)
  passencoder.setBindGroup(0, mvp2group)
  passencoder.draw(cube.vertexCount)
  passencoder.end()
  // webgpu run in a separate process, all the commands will be executed after submit
  device.queue.submit([encoder.finish()])
}

async function run() {
  const canvas = document.querySelector('canvas')
  if (!canvas)
    throw new Error('no canvas')
  const { adapter, device, context, format } = await initWebGPU(canvas)
  const { Pipeline, vertexbuffer, uniformgroup, mvpbuffer, mvp2group } = await initPipeline(device, format)
  //获取贴图
  const res = await fetch(textureUrl)
  const img = await res.blob()
  const bitmap = await createImageBitmap(img)
  const textureSize = [bitmap.width, bitmap.height]
  //创建材质
  const texture = device.createTexture({
    size: textureSize,
    format: 'rgba8unorm',
    usage:
      GPUTextureUsage.TEXTURE_BINDING |
      GPUTextureUsage.COPY_DST |
      GPUTextureUsage.RENDER_ATTACHMENT
  })
  // update image to GPUTexture

  // Create a sampler with linear filtering for smooth interpolation.线性采样
  const sampler = device.createSampler({
    // addressModeU: 'repeat',
    // addressModeV: 'repeat',
    magFilter: 'linear',
    minFilter: 'linear'
  })
  const textureGroup = device.createBindGroup({
    label: 'Texture Group with Texture/Sampler',
    layout: Pipeline.getBindGroupLayout(1),
    entries: [
      {
        binding: 0,
        resource: sampler
      },
      {
        binding: 1,
        resource: texture.createView()
      }
    ]
  })

  device.queue.copyExternalImageToTexture(
    { source: bitmap },
    { texture: texture },
    textureSize
  )

  draw(device, context, Pipeline, vertexbuffer, uniformgroup, mvp2group,textureGroup)
  // default state

  // start loop
  function frame() {
    // rotate by time, and update transform matrix
    const now = Date.now() / 1000
    rotation.x = Math.sin(now)
    rotation.y = Math.cos(now)
    const mvpMatrix = math.getMvpMatrix(aspect, position, rotation, scale)
    const mvpMatrix2 = math.getMvpMatrix(aspect, position2, rotation, scale)
    device.queue.writeBuffer(
      mvpbuffer,
      0,
      mvpMatrix
    )
    device.queue.writeBuffer(
      mvpbuffer,
      256,
      mvpMatrix2
    )
    // then draw
    draw(device, context, Pipeline, vertexbuffer, uniformgroup, mvp2group,textureGroup)
    requestAnimationFrame(frame)
  }
  frame()
}
</script>






<style scoped>
* {
  margin: 0;
  padding: 0;
  overflow: hidden;
}

#canvas {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 0;
}
</style>
