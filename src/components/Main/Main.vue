<template>
  <header>
    <section class="upload-list">
      <div class="item">
        <t-input-adornment>
          <template #append>
            <t-upload
              accept="image/*"
              :showUploadProgress="false"
              :useMockProgress="false"
              :autoUpload="false"
              theme="custom"
              :onChange="alipayChange"
              :trigger-button-props="{ theme: 'primary', variant: 'base' }"
            />
          </template>
          <t-input
            :status="alipayVal ? 'success' : ''"
            readonly
            v-model="alipayVal"
            placeholder="请上传 支付宝 收款码"
          />
        </t-input-adornment>
      </div>
      <div class="item">
        <t-input-adornment>
          <template #append>
            <t-upload
              accept="image/*"
              :showUploadProgress="false"
              :useMockProgress="false"
              :autoUpload="false"
              theme="custom"
              :onChange="wechatChange"
              :trigger-button-props="{ theme: 'success', variant: 'base' }"
            />
          </template>
          <t-input
            :status="wechatVal ? 'success' : ''"
            readonly
            v-model="wechatVal"
            placeholder="请上传 微信 收款码"
          />
        </t-input-adornment>
      </div>
      <div class="item flex">
        <t-radio-group size="small" v-model="themeStatus">
          <t-radio-button value="none">默认</t-radio-button>
          <t-radio-button value="theme_a">主题A</t-radio-button>
          <t-radio-button value="theme_b">主题B</t-radio-button>
          <t-radio-button value="theme_c">主题C</t-radio-button>
          <t-radio-button value="theme_d">主题D</t-radio-button>
        </t-radio-group>
      </div>
    </section>
    <t-card class="alipay-hide">
      <span class="title">缺省比例</span>
      <t-slider v-model="qrHideSize" :onChange="alipayHideChange" />
    </t-card>
  </header>
  <main :class="{ show: wechatQrcodeImg && alipayQrcodeImg }">
    <t-loading :loading="isLoading" />
    <div class="qr-main" :class="themeStatus" ref="qrcodeDom" v-if="!isLoading">
      <img class="wechat-qr" :src="wechatQrcodeImg" crossorigin="anonymous" />
      <img class="alipay-qr" :src="alipayQrcodeImg" crossorigin="anonymous" />
    </div>
    <t-button theme="success" size="large" class="download-qrcode" :onClick="downloadQrcode" :disabled="isLoading || !wechatQrcodeImg || !alipayQrcodeImg">
      <template #icon>
        <svg
          xmlns="http://www.w3.org/2000/svg"
          width="24"
          height="24"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
          stroke-linecap="round"
          stroke-linejoin="round"
        >
          <path stroke="none" d="M0 0h24v24H0z" fill="none" />
          <path d="M19 18a3.5 3.5 0 0 0 0 -7h-1a5 4.5 0 0 0 -11 -2a4.6 4.4 0 0 0 -2.1 8.4" />
          <path d="M12 13l0 9" />
          <path d="M9 19l3 3l3 -3" />
        </svg>
      </template>
      下载二维码
    </t-button>
  </main>
  
  <!-- 主题预览对话框 -->
  <t-dialog
    v-model:visible="isThemePreviewVisible"
    header="主题预览"
    :footer="false"
    width="80%"
    :close-on-overlay-click="true"
  >
    <div class="theme-preview-container">
      <div class="theme-item">
        <div class="theme-preview none"></div>
        <div class="theme-name">默认</div>
      </div>
      <div class="theme-item">
        <div class="theme-preview theme_a"></div>
        <div class="theme-name">主题A</div>
      </div>
      <div class="theme-item">
        <div class="theme-preview theme_b"></div>
        <div class="theme-name">主题B</div>
      </div>
      <div class="theme-item">
        <div class="theme-preview theme_c"></div>
        <div class="theme-name">主题C</div>
      </div>
      <div class="theme-item">
        <div class="theme-preview theme_d"></div>
        <div class="theme-name">主题D</div>
      </div>
    </div>
  </t-dialog>
</template>
<script lang="ts" setup>
import { ref, onMounted } from 'vue'
import html2canvas from 'html2canvas'
import jsQR from 'jsqr'
import QrCode from 'qrcode'
import { NotifyPlugin } from 'tdesign-vue-next'

// 添加加载状态
const isLoading = ref<boolean>(false)

//  是否美化
const themeStatus = ref<string>('none')

//  上传支付宝收款码
const alipayFileList = ref<any>()
const alipayVal = ref<any>()
const alipayQrcodeImg = ref<string>()
// 支付宝收款码白名单
const alipayWhiteList = ['alipay.com']
const alipayChange = async (v: any) => {
  try {
    isLoading.value = true
    alipayFileList.value = v && v.length ? v[0].raw : alipayFileList.value
    if (!alipayFileList.value) {
      isLoading.value = false
      return
    }
    const res = await loadImg(alipayFileList.value)
    if (!res || !alipayWhiteList.some((i: string) => String(res).toLowerCase().includes(i))) {
      isLoading.value = false
      return NotifyPlugin.error({
        title: 'Error',
        content: '请上传正确的 支付宝 收款码',
      })
    }
    alipayVal.value = String(res)
    alipayQrcodeImg.value = await drawQR(alipayVal.value, true)
    showQrcode()
  } catch (error) {
    NotifyPlugin.error({
      title: 'Error',
      content: '处理支付宝收款码时出错，请重试',
    })
  } finally {
    isLoading.value = false
  }
}

const wechatChange = async (v: any) => {
  try {
    isLoading.value = true
    wechatFileList.value = v && v.length ? v[0].raw : wechatFileList.value
    if (!wechatFileList.value) {
      isLoading.value = false
      return
    }
    const res = await loadImg(wechatFileList.value)
    if (!res || !wechatWhiteList.some((i: string) => String(res).toLowerCase().includes(i))) {
      isLoading.value = false
      return NotifyPlugin.error({
        title: 'Error',
        content: '请上传正确的 微信 收款码',
      })
    }
    wechatVal.value = String(res)
    wechatQrcodeImg.value = await drawQR(wechatVal.value)
    showQrcode()
  } catch (error) {
    NotifyPlugin.error({
      title: 'Error',
      content: '处理微信收款码时出错，请重试',
    })
  } finally {
    isLoading.value = false
  }
}

//  识别二维码
const qrcodeResult = (image: any) => {
  const canvas = document.createElement('canvas')
  const context: any = canvas.getContext('2d')
  canvas.width = image.width
  canvas.height = image.height
  context.drawImage(image, 0, 0, canvas.width, canvas.height)
  
  // 尝试不同的图像处理方法来提高识别率
  const imageData = context.getImageData(0, 0, canvas.width, canvas.height)
  let decodedResult = jsQR(imageData.data, imageData.width, imageData.height, {
    inversionAttempts: 'dontInvert',
  })
  
  // 如果第一次识别失败，尝试调整对比度后再次识别
  if (!decodedResult) {
    // 调整对比度
    const data = imageData.data
    const contrast = 1.5 // 对比度调整参数
    const factor = (259 * (contrast + 255)) / (255 * (259 - contrast))
    
    for (let i = 0; i < data.length; i += 4) {
      data[i] = factor * (data[i] - 128) + 128
      data[i + 1] = factor * (data[i + 1] - 128) + 128
      data[i + 2] = factor * (data[i + 2] - 128) + 128
    }
    
    context.putImageData(imageData, 0, 0)
    const enhancedImageData = context.getImageData(0, 0, canvas.width, canvas.height)
    
    decodedResult = jsQR(enhancedImageData.data, enhancedImageData.width, enhancedImageData.height, {
      inversionAttempts: 'attemptBoth',
    })
  }
  
  return decodedResult?.data
}

// 添加清除右上角的函数
const qrcodeDom = ref<any>()
const qrSize = ref<number>(0)
const drawQR = async (qrValue: string, hide: boolean = false) => {
  const qrDom = document.createElement('canvas')
  await QrCode.toCanvas(qrDom, qrValue, {
    errorCorrectionLevel: 'H',
    width: qrSize.value,
    margin: 0,
    color: {
      dark: '#000000',
      light: '#ffffff',
    },
  })
  if (hide) {
    const ctx: any = qrDom.getContext('2d')
    const clearWidth = (qrSize.value / 2) * (qrHideSize.value / 100)
    ctx.clearRect(qrSize.value / 2, qrSize.value - clearWidth, qrSize.value / 2, clearWidth)
  }
  return qrDom.toDataURL('image/png')
}

// 下载二维码
const downloadQrcode = async () => {
  const canvas = await html2canvas(qrcodeDom.value, { useCORS: true })
  const dataURL = canvas.toDataURL('image/png')
  const link = document.createElement('a')
  link.href = dataURL
  link.download = 'PayQrcode.png'
  link.click()
}

onMounted(() => {
  qrSize.value = qrcodeDom.value!.clientWidth
})
</script>
<style lang="less" scoped>
@import url('./Main.less');

.theme-preview-container {
  display: flex;
  flex-wrap: wrap;
  gap: 20px;
  justify-content: center;
  
  .theme-item {
    display: flex;
    flex-direction: column;
    align-items: center;
    
    .theme-preview {
      width: 120px;
      height: 120px;
      border: 1px solid #eee;
      border-radius: 8px;
      overflow: hidden;
      
      &.none {
        background: #fff;
      }
      
      &.theme_a {
        background: url('../../assets/images/theme_a.webp') no-repeat;
        background-size: contain;
      }
      
      &.theme_b {
        background: url('../../assets/images/theme_b.webp') no-repeat;
        background-size: contain;
      }
      
      &.theme_c {
        background: url('../../assets/images/theme_c.webp') no-repeat;
        background-size: contain;
      }
      
      &.theme_d {
        background: url('../../assets/images/theme_d.webp') no-repeat;
        background-size: contain;
      }
    }
    
    .theme-name {
      margin-top: 8px;
      font-size: 14px;
    }
  }
}
</style>
