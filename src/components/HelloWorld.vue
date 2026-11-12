<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

// krpano 实例接口（embedpano 的 onready 回调中注入）
const krpano = ref(null)
// 当前拖拽/编辑中的热点名称
const portName = ref(null)
// 鼠标跟踪相关状态
const trackMouseEnabled = ref(false)
const trackMouseId = ref(null)
const trackMouseValue = ref(null)

// krpano 就绪回调
const krpanoOnready = krpano_interface => {
  krpano.value = krpano_interface
  const a = krpano_interface.get('global')
  console.log(a)
}

// krpano 错误回调
const krpanoOnerror = error => {
  console.log(error)
}

// 当前位置添加热点
const addHotsport = () => {
  if (!krpano.value) return
  // 以当前视角为中心创建热点
  const h = krpano.value.get('view.hlookat')
  const v = krpano.value.get('view.vlookat')
  const hs_name = 'hs' + ((Date.now() + Math.random()) | 0)
  portName.value = hs_name

  krpano.value.call('addhotspot(' + hs_name + ');')
  // 相对页面路径（vtour/ 位于 public 目录），兼容 GitHub Pages 子路径部署
  krpano.value.set('hotspot[' + hs_name + '].url', 'vtour/skin/vtourskin_hotspot.png')
  krpano.value.set('hotspot[' + hs_name + '].ath', h)
  krpano.value.set('hotspot[' + hs_name + '].atv', v)
  krpano.value.set('hotspot[' + hs_name + '].distorted', true)
  krpano.value.set('hotspot[' + hs_name + '].scale', 0.7)
  krpano.value.set('hotspot[' + hs_name + '].dragging', true)
  krpano.value.set('hotspot[' + hs_name + '].ondown', 'draghotspot()')
  // krpano 的 set() 不支持直接传 JS 函数（会报 Unknown action），
  // 需通过对象属性直接赋值来绑定事件
  const hs = krpano.value.get('hotspot[' + hs_name + ']')
  if (hs) {
    hs.onclick = function () {
      console.log('hotspot "' + hs_name + '" clicked, 球面坐标: ath=' + h + ', atv=' + v)
    }
  }
  console.log('已添加热点:', hs_name)
}

// 移除所有热点
const removeHotsport = () => {
  if (!krpano.value) return
  krpano.value.call('loop(hotspot.count GT 0, removehotspot(0) );')
}

// 加载 XML 字符串
const loadXmlString = () => {
  if (!krpano.value) return
  const xmlstring = `<krpano>
                            <preview type="grid(cube,64,64,512,0xCCCCCC,0xF6F6F6,0x999999);" />
                            <view hlookat="0" vlookat="0" fov="100" distortion="0.0" />
                         </krpano>`
  krpano.value.call('loadxml(' + encodeURIComponent(xmlstring) + ', null, MERGE, BLEND(0.5));')
}

// 加载全景 tour.xml
const loadPano = () => {
  if (!krpano.value) return
  /** loadpano(xmlpath, vars, flags, blend, loaddone) */
  krpano.value.call('loadpano(tour.xml, null, MERGE, BLEND(0.5));')
}

// 设置随机视图
const setRandomView = () => {
  if (!krpano.value) return
  krpano.value.set('view.hlookat', Math.random() * 360.0 - 180.0)
  krpano.value.set('view.vlookat', Math.random() * 180.0 - 90.0)
  krpano.value.set('view.fov', 80.0 + Math.random() * 100.0)
  krpano.value.set('view.distortion', Math.random() * 1.0)
}

// 动画查看随机视图
const lookToRandomView = () => {
  if (!krpano.value) return
  krpano.value.call('lookto(' + (Math.random() * 360.0 - 180.0) + ',' + (Math.random() * 180.0 - 90.0) + ',' + (80.0 + Math.random() * 100.0) + ')')
}

// 获取当前视图
const getCurrentView = () => {
  if (!krpano.value) return
  const hlookat = krpano.value.get('view.hlookat')
  const vlookat = krpano.value.get('view.vlookat')
  const fov = krpano.value.get('view.fov')
  const distortion = krpano.value.get('view.distortion')
  console.log(
    'hlookat', hlookat.toFixed(2), '\n',
    'vlookat', vlookat.toFixed(2), '\n',
    'fov', fov.toFixed(2), '\n',
    'distortion', distortion.toFixed(2)
  )
}

// 每帧更新鼠标对应的球面坐标
const mouseChange = () => {
  if (!krpano.value) return
  const mx = krpano.value.get('mouse.x')
  const my = krpano.value.get('mouse.y')
  const pnt = krpano.value.screentosphere(mx, my)
  trackMouseValue.value = {
    x: mx.toFixed(0),
    y: my.toFixed(0),
    ath: pnt.x.toFixed(2),
    atv: pnt.y.toFixed(2)
  }
}

// 开始 / 停止跟踪鼠标位置
const trackMouse = () => {
  if (!krpano.value) return
  if (!trackMouseEnabled.value) {
    trackMouseId.value = setInterval(mouseChange, 1000.0 / 60.0)
    trackMouseEnabled.value = true
  } else {
    clearInterval(trackMouseId.value)
    trackMouseId.value = null
    trackMouseValue.value = null
    trackMouseEnabled.value = false
  }
}

// 修改热点
const modifyHotsport = () => {
  if (!krpano.value) return
  if (!portName.value) {
    console.warn('暂无热点可编辑，请先添加热点')
    return
  }
  // krpano 的 set() 不支持直接传 JS 函数，改用对象属性直接赋值绑定 ondown
  const hs = krpano.value.get('hotspot[' + portName.value + ']')
  if (!hs) return
  hs.ondown = function () {
    const mx = krpano.value.get('mouse.x')
    const my = krpano.value.get('mouse.y')
    const pnt = krpano.value.screentosphere(mx, my)
    console.log('热点球面坐标:', pnt)
  }
}

onMounted(() => {
  embedpano({
    // 相对路径（vtour/ 在 public 目录，构建时复制到 dist 根），
    // 相对当前页面解析，兼容 GitHub Pages 子路径部署
    swf: 'vtour/tour.swf',
    xml: 'vtour/tour.xml',
    target: 'pano',
    html5: 'auto',
    mobilescale: 1.0,
    passQueryParameters: true,
    onready: krpanoOnready,
    onerror: krpanoOnerror,
    consolelog: true
  })
})

// 组件卸载时清理鼠标跟踪定时器，避免内存泄漏
onUnmounted(() => {
  if (trackMouseId.value) {
    clearInterval(trackMouseId.value)
    trackMouseId.value = null
  }
})
</script>

<template>
  <button @click="addHotsport">当前位置添加热点</button>
  <button @click="removeHotsport">移除所有热点</button>
  <button @click="loadXmlString">加载xml字符串</button>
  <button @click="loadPano">加载全景</button>
  <button @click="setRandomView">设置随机视图</button>
  <button @click="lookToRandomView">动画查看随机视图</button>
  <button @click="getCurrentView">获取当前视图</button>
  <button @click="modifyHotsport">编辑热点</button>
  <button @click="trackMouse">跟踪鼠标位置</button>
  <span v-if="trackMouseValue" class="mouse-info">
    屏幕({{ trackMouseValue.x }}, {{ trackMouseValue.y }}) →
    球面({{ trackMouseValue.ath }}, {{ trackMouseValue.atv }})
  </span>
  <div class="container">
    <div id="pano"></div>
  </div>
</template>

<style scoped>
  .container {
    width: 100%;
    height: 100%;
    position: absolute;
  }
  #pano {
    width: 100%;
    height: 90%;
  }
  .mouse-info {
    font-family: monospace;
    font-size: 12px;
    margin-left: 12px;
  }
</style>
