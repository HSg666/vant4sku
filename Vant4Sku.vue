<template>
	<van-popup v-model:show="modelVisible" round closeable position="bottom" @update:show="onPopupClose" safe-area-inset-bottom class="vant4-sku-selector" style="max-height: 80%; min-height: 50%">
		<div class="vant4-sku-popup">
			<!-- <p v-if="!selectedSku && product.skulist.every(sku => sku.stock === 0)" style="color: #ff9719">该商品暂时无货，敬请期待补货通知！</p> -->

			<!-- 商品头部 -->
			<div class="vant4-popup-header" @click="openImagePreview">
				<!-- sku图片区 -->
				<div style="position: relative" v-if="showImage">
					<img :src="selectedSku?.img || product.mainImage" alt="" class="vant4-popup-img" />
					<van-icon name="enlarge" size="14" class="vant4-popup-icon" />
				</div>
				<div class="vant4-popup-info">
					<p class="vant4-popup-price">
						<!-- 优惠价 -->
						<span v-if="selectedSku && selectedSku.stock > 0">
							<span style="font-size: 14px">¥</span>
							{{ Number(selectedSku?.price).toFixed(2) }}
						</span>
						<!-- 无有效 SKU 或库存不足 -->
						<span v-else style="color: #ffad00; font-size: 16px">请选择规格</span>

						<!-- 插槽优先 -->
						<span v-if="showOriginPrice && $slots['origin-price']" class="vant4-slot-origin-price">
							<slot name="origin-price" :sku="selectedSku" />
						</span>

						<!-- 默认划线价（兜底，默认显示） -->
						<span v-else-if="showOriginPrice && selectedSku?.originPrice" class="vant4-default-origin-price">¥{{ selectedSku.originPrice.toFixed(2) }}</span>
					</p>
					<p class="vant4-popup-stock" v-if="showStock">剩余 {{ selectedSku?.stock }} 件</p>
					<p class="vant4-popup-selected">
						已选：
						<span v-for="(value, key, index) in selectedSpecs" :key="key">
							{{ value }}
							<span v-if="index < Object.keys(selectedSpecs).length - 1">,</span>
						</span>
					</p>
					<p class="vant4-popup-stock" v-if="showProductCode">商品编号：{{ selectedSku?.productCode }}</p>
				</div>
			</div>

			<!-- 内容区  -->
			<div class="vant4-sku-body">
				<!-- 动态规格选择 -->
				<div v-for="spec in product.specs" :key="spec.key" class="vant4-spec-group">
					<div class="vant4-spec-title">{{ spec.name }}</div>
					<div class="vant4-spec-options">
						<van-tag
							v-for="option in spec.options"
							:key="option"
							:color="getOptionState(spec.key, option).disabled ? '#f7f8fa' : getOptionState(spec.key, option).selected ? '#fde6e9' : '#f7f8fa'"
							:text-color="getOptionState(spec.key, option).disabled ? '#c8c9cc' : getOptionState(spec.key, option).selected ? '#ee0a24' : '#323233'"
							@click="selectOption(spec.key, option)"
							round
							size="large"
							:disabled="getOptionState(spec.key, option).disabled"
							class="vant4-spec-tag"
						>
							{{ option }}
						</van-tag>
					</div>
				</div>

				<!-- 分期选择插槽 -->
				<div v-if="$slots['select-installment']" class="vant4-installment-group">
					<div class="vant4-installment-title">
						{{ installmentTitle }}
						<span v-if="showInstallmentTips" class="vant4-installment-tips">
							{{ installmentTips }}
						</span>
					</div>
					<van-radio-group v-model="checkedInstallment" direction="horizontal">
						<van-radio v-for="item in installmentOptions" :key="item" :name="item" icon-size="14" shape="square" class="vant4-installment-radio">
							<span v-if="item === '0'" style="font-size: 13px">不分期</span>
							<!-- 修复：只有选中有效 SKU 时才计算分期价 -->
							<span v-else-if="selectedSku && selectedSku.price" style="font-size: 13px">{{ (selectedSku.price / item).toFixed(2) }} ✖{{ item }}期</span>
							<!-- 未选中时显示占位 -->
							<span v-else style="font-size: 13px">-- ✖{{ item }}期</span>
						</van-radio>
					</van-radio-group>
				</div>

				<!-- 数量选择 -->
				<div class="vant4-quantity-group">
					<span class="vant4-quantity-label">{{ quantityText }}</span>
					<div class="vant4-quota-group">
						<span class="vant4-quota-text" v-if="showQuotaText">({{ finalMin }}件起售，限购{{ finalMax }}件)&nbsp;&nbsp;</span>

						<!-- 只支持整数 -->
						<van-stepper v-model="quantity" :min="finalMin" :max="finalMax" input-width="50" button-size="20" />
					</div>
				</div>

				<!-- 备用的文本框 -->
				<van-cell-group>
					<!-- 输入手机号 -->
					<van-field v-if="showTel" v-model="tel" type="tel" label="手机号" maxlength="11" placeholder="请输入手机号" />
					<!-- 输入邮箱 -->
					<van-field v-if="showEmail" v-model="email" type="text" label="邮箱" placeholder="请输入邮箱" />
					<!-- 留言输入框 -->
					<van-field v-if="showMessages" v-model="message" label="留言" type="text" placeholder="请输入留言" autosize />

					<!-- 日历选择器 -->
					<van-cell v-if="showDate" title="日期" :value="date" @click="isShowDate = true" is-link />
					<van-calendar v-model:show="isShowDate" @confirm="DateonConfirm" />

					<!-- 选择器 -->
					<van-field v-if="showPicker" v-model="pickerViewValue" is-link readonly name="picker" :label="selectTitle" :placeholder="selectTips" @click="showPickerPop = true" />
					<van-popup v-model:show="showPickerPop" destroy-on-close position="bottom">
						<van-picker :columns="pickerColumns" :model-value="pickerValue" @confirm="PickeronConfirm" @cancel="showPickerPop = false" />
					</van-popup>

					<!-- 备注输入框 -->
					<van-field v-if="showRemark" v-model="remark" label="备注" type="textarea" maxlength="50" placeholder="请输入备注" autosize />
				</van-cell-group>

				<!-- 备用底部插槽 -->
				<div class="vant4-spec-group">
					<slot name="footer"></slot>
				</div>
			</div>

			<!-- 操作按钮 -->
			<div class="vant4-action-buttons">
				<van-action-bar-button
					type="warning"
					:text="cartButtonText"
					@click="addToCart"
					:disabled="!selectedSku || selectedSku.stock <= 0"
					v-if="props.showAddCartBtn && selectedSku?.stock > 0"
				/>
				<van-action-bar-button type="danger" :text="buyButtonText" @click="buyNow" :disabled="!selectedSku || selectedSku.stock <= 0" v-if="selectedSku?.stock > 0" />

				<!-- 如果没库存，那么就不显示加入购物车按钮 -->
				<van-action-bar-button v-else type="danger" text="库存不足" :disabled="!selectedSku || selectedSku.stock <= 0" />
			</div>
		</div>
	</van-popup>

	<van-image-preview v-model:show="showImagePreviewPopup" :images="[previewImageUrl]" :closeable="true" />
</template>

<script setup>
import { ref, computed, reactive, watch, onMounted, nextTick } from 'vue'
import { Button, Popup, Tag, Stepper, showImagePreview, showToast } from 'vant'

const props = defineProps({
	// 数据源
	product: {
		type: Object,
		required: true,
	},
	// 默认显示sku选择器
	modelValue: {
		type: Boolean,
		default: false,
	},
	// 传入的skuid
	defaultSkuId: {
		type: Number,
		default: null,
	},
	// 默认显示加入购物车按钮
	showAddCartBtn: {
		type: Boolean,
		default: true,
	},
	// 是否显示图片（默认显示）
	showImage: {
		type: Boolean,
		default: true,
	},
	// 默认隐藏手机号输入框
	showTel: {
		type: Boolean,
		default: false,
	},
	// 默认隐藏邮箱输入框
	showEmail: {
		type: Boolean,
		default: false,
	},
	// 默认隐藏留言框
	showMessages: {
		type: Boolean,
		default: false,
	},
	// 默认显示备注输入框
	showRemark: {
		type: Boolean,
		default: false,
	},
	// 默认显示日历
	showDate: {
		type: Boolean,
		default: false,
	},
	// 默认显示选择器
	showPicker: {
		type: Boolean,
		default: false,
	},
	// 选择器的列数
	columns: {
		type: Array,
		default: () => [],
	},
	// 显示原价
	showOriginPrice: { type: Boolean, default: true },
	// 显示库存
	showStock: { type: Boolean, default: true },
	// 显示商品编码
	showProductCode: { type: Boolean, default: true },

	cartButtonText: { type: String, default: '加入购物车' },
	buyButtonText: { type: String, default: '立即购买' },
	selectTitle: { type: String, default: '选择器' },
	selectTips: { type: String, default: '点击选择城市' },
	// 数量选择器最小值
	stepperMin: { type: Number, default: 1 },
	// 数量选择器最大值
	stepperMax: { type: Number, default: null }, // null 表示不限制最大值（默认用库存）
	// 是否显示限购提示
	showQuotaText: { type: Boolean, default: false },
	// 购买数量文字
	quantityText: { type: String, default: '购买数量' },

	// 分期选项
	installmentOptions: {
		type: Array,
		default: () => [],
	},
	// 分期标题
	installmentTitle: {
		type: String,
		default: '选择分期',
	},
	// 分期提示语
	installmentTips: {
		type: String,
		default: '(*分期限龙卡信用卡支付)',
	},
	// 是否显示分期提示语
	showInstallmentTips: {
		type: Boolean,
		default: false,
	},
})

// 暴露出去的自定义事件
const emit = defineEmits(['addToCart', 'buyNow', 'changeStepper', 'update:modelValue', 'close'])

// 内部状态控制弹窗的显隐
const modelVisible = ref(props.modelValue)

// 监听外部 modelValue 变化同步到 modelVisible
watch(
	() => props.modelValue,
	newVal => {
		modelVisible.value = newVal
	},
)

// 当前选择的规格（动态）
const selectedSpecs = reactive({})

// 购买数量  --------------------  start
const quantity = ref(1)
// 监听购买数量变化
watch(quantity, newVal => {
	emit('changeStepper', newVal)
})

// 计算 stepper 的实际最小值和最大值
const stepperMinComputed = computed(() => {
	return props.stepperMin >= 1 ? props.stepperMin : 1
})

// 计算 stepper 的实际最大值
const stepperMaxComputed = computed(() => {
	const stock = selectedSku.value?.stock || 1
	const max = props.stepperMax || stock
	return Math.min(max, stock)
})

// 确保 min <= max
const finalMin = computed(() => {
	return stepperMinComputed.value <= stepperMaxComputed.value ? stepperMinComputed.value : 1
})

const finalMax = computed(() => {
	return stepperMaxComputed.value >= finalMin.value ? stepperMaxComputed.value : finalMin.value
})
// 购买数量  --------------------  end

// 分期选择
const checkedInstallment = ref(props.installmentOptions[0] || '0')

// 默认隐藏大图预览
const showImagePreviewPopup = ref(false)
// 当前预览的图片
const previewImageUrl = ref('')

// 手机号
const tel = ref('')
// 邮箱
const email = ref('')

// 留言内容
const message = ref('')
// 备注内容
const remark = ref('')
// 日期
const date = ref('请选择时间')
// 日期弹窗显示
const isShowDate = ref(false)

// 当前主图
const currentMainImg = computed(() => {
	return selectedSku.value?.img || props.product.mainImage
})

// 获取当前选中的 SKU
const selectedSku = computed(() => {
	return (
		props.product.skulist.find(sku => {
			return Object.keys(selectedSpecs).every(key => sku[key] === selectedSpecs[key])
		}) || null
	)
})

// 获取规格标题（用于展示）
const getSpecTitle = key => {
	const spec = props.product.specs.find(s => s.key === key)
	return spec ? spec.name : key
}

// 获取当前可选的规格选项（用于禁用判断）
const getAvailableOptions = (specKey, selectedSpecsCopy) => {
	return props.product.skulist
		.filter(sku => {
			return Object.keys(selectedSpecsCopy).every(key => {
				if (key === specKey) return true // 排除自己
				return !selectedSpecsCopy[key] || sku[key] === selectedSpecsCopy[key]
			})
		})
		.map(sku => sku[specKey])
}

// 缓存每个规格选项的禁用状态（优化性能）
const optionDisabledCache = new Map()
const getCacheKey = (specKey, option) => `${specKey}:${option}:${JSON.stringify(selectedSpecs)}`

// 判断某规格选项是否禁用（无库存或无法组合）- 优化版（带缓存）
const isOptionDisabled = (specKey, option) => {
	const cacheKey = getCacheKey(specKey, option)
	if (optionDisabledCache.has(cacheKey)) {
		return optionDisabledCache.get(cacheKey)
	}

	// 1. 检查是否有库存
	const hasStock = props.product.skulist.some(sku => {
		return sku[specKey] === option && sku.stock > 0
	})
	if (!hasStock) {
		optionDisabledCache.set(cacheKey, true)
		return true
	}

	// 2. 检查是否能组合出有效 SKU
	const selectedSpecsCopy = { ...selectedSpecs }
	selectedSpecsCopy[specKey] = option
	const availableOptions = getAvailableOptions(specKey, selectedSpecsCopy)
	const isDisabled = !availableOptions.includes(option)
	optionDisabledCache.set(cacheKey, isDisabled)
	return isDisabled
}

// 预计算所有规格选项的状态（优化渲染性能）
const specOptionsState = computed(() => {
	const state = {}
	props.product.specs.forEach(spec => {
		state[spec.key] = {}
		spec.options.forEach(option => {
			state[spec.key][option] = {
				disabled: isOptionDisabled(spec.key, option),
				selected: selectedSpecs[spec.key] === option,
			}
		})
	})
	return state
})

// 获取选项状态（从预计算的 computed 中获取）
const getOptionState = (specKey, option) => {
	return specOptionsState.value[specKey]?.[option] || { disabled: false, selected: false }
}

// 监听 selectedSpecs 变化时清除缓存
watch(
	() => selectedSpecs,
	() => {
		optionDisabledCache.clear()
	},
	{ deep: true },
)

// 选择规格（支持取消选择）
const selectOption = (specKey, option) => {
	// 如果点击已选中的选项，则取消选择
	if (selectedSpecs[specKey] === option) {
		selectedSpecs[specKey] = ''
		// 清除缓存以便重新计算
		optionDisabledCache.clear()
	} else {
		// 检查是否可选
		if (isOptionDisabled(specKey, option)) return
		selectedSpecs[specKey] = option
		// 清除缓存以便重新计算
		optionDisabledCache.clear()
	}
}

// 打开图片预览
const openImagePreview = () => {
	const url = selectedSku.value?.img || props.product.mainImage
	previewImageUrl.value = url
	showImagePreview({
		images: [url],
		closeable: true,
	})
	showImagePreviewPopup.value = false
}

// 选择器

// 选择器的列数
const pickerColumns = ref(props.columns)
watch(
	() => props.columns,
	newVal => {
		pickerColumns.value = newVal
	},
)
const pickerViewValue = ref('') // 页面显示选中值
const pickerValue = ref([]) // 选中对象的value
const pickerResult = ref({}) // 选中的对象

const showPickerPop = ref(false) // 选择器弹窗显隐
// 选择器确认回调
const PickeronConfirm = ({ selectedValues, selectedOptions }) => {
	pickerViewValue.value = selectedOptions[0]?.text
	pickerValue.value = selectedValues
	pickerResult.value = selectedOptions[0]
	showPickerPop.value = false
	console.log(pickerResult.value, 'pickerResult.value')
}
// 选择器

// 加入购物车
const addToCart = () => {
	if (!selectedSku.value || selectedSku.value.stock <= 0) {
		showToast('库存不足，无法添加')
		return
	}
	// 仅当手机号显示时才校验
	if (props.showTel && (!tel.value || !/^\d{11}$/.test(tel.value))) {
		showToast('请输入正确的手机号')
		return
	}
	// 可选：邮箱格式校验，有显示没值要提示用户输入邮箱
	if (props.showEmail) {
		if (!email.value) {
			showToast('请输入邮箱')
			return
		}
		if (email.value && !/^\S+@\S+\.\S+$/.test(email.value)) {
			showToast('请输入正确的邮箱')
			return
		}
	}
	// 日期有显示并没值
	if (props.showDate && date.value == '请选择时间') {
		showToast('请选择时间')
		return
	}
	// 选择器有显示但是没值就是提示用户
	if (props.showPicker && (!pickerValue.value || Object.keys(pickerValue.value).length === 0)) {
		showToast('请选择')
		return
	}

	// 响应体
	const result = {
		sku: selectedSku.value,
		quantity: quantity.value,
		// 补充 product 中除了 skulist 和 specs 之外的数据
		...Object.fromEntries(Object.entries(props.product).filter(([key]) => !['skulist', 'specs'].includes(key))),
	}

	// 只在使用分期插槽时返回分期值
	if (props.installmentOptions.length > 0 && props.installmentOptions.some(item => item !== undefined)) {
		result.installment = checkedInstallment.value
	}

	// 根据显隐判断是否返回给使用者，若不显示则不返回
	if (props.showTel) result.tel = tel.value
	if (props.showEmail) result.email = email.value
	if (props.showMessages) result.message = message.value
	if (props.showRemark) result.remark = remark.value
	if (props.showDate) result.date = date.value
	if (props.showPicker) result.pickerResult = pickerResult.value // 选择器

	emit('addToCart', result)
	closePopup()
}

// 立即购买
const buyNow = () => {
	if (!selectedSku.value || selectedSku.value.stock <= 0) {
		showToast('库存不足，无法购买')
		return
	}
	// 仅当手机号显示时才校验
	if (props.showTel && (!tel.value || !/^\d{11}$/.test(tel.value))) {
		showToast('请输入正确的手机号')
		return
	}
	// 可选：邮箱格式校验，有显示没值要提示用户输入邮箱
	if (props.showEmail) {
		if (!email.value) {
			showToast('请输入邮箱')
			return
		}
		if (email.value && !/^\S+@\S+\.\S+$/.test(email.value)) {
			showToast('请输入正确的邮箱')
			return
		}
	}
	// 这个为何没校验，有显示但没值需要提示
	if (props.showDate && date.value == '请选择时间') {
		showToast('请选择时间')
		return
	}
	// 选择器有显示但是没值就是提示用户
	if (props.showPicker && (!pickerValue.value || Object.keys(pickerValue.value).length === 0)) {
		showToast('请选择')
		return
	}

	// 响应体
	const result = {
		sku: selectedSku.value,
		quantity: quantity.value,
		// 补充 product 中除了 skulist 和 specs 之外的数据
		...Object.fromEntries(Object.entries(props.product).filter(([key]) => !['skulist', 'specs'].includes(key))),
	}
	// 只在使用分期插槽时返回分期值
	if (props.installmentOptions.length > 0 && props.installmentOptions.some(item => item !== undefined)) {
		result.installment = checkedInstallment.value
	}

	// 根据显隐判断是否返回给使用者，若不显示则不返回
	if (props.showTel) result.tel = tel.value
	if (props.showEmail) result.email = email.value
	if (props.showMessages) result.message = message.value
	if (props.showRemark) result.remark = remark.value
	if (props.showDate) result.date = date.value
	if (props.showPicker) result.pickerResult = pickerResult.value // 选择器

	emit('buyNow', result)

	closePopup()
}

// 关闭弹窗
const closePopup = () => {
	emit('update:modelValue', false)
	emit('close')
}

// 弹窗关闭回调
const onPopupClose = visibleState => {
	if (!visibleState) closePopup()
}

// 初始化默认选中
const initDefaultSelection = () => {
	const skuIdFromQuery = props.defaultSkuId
	if (skuIdFromQuery) {
		const matchedSku = props.product.skulist.find(sku => sku.id === skuIdFromQuery && sku.stock > 0)
		if (matchedSku) {
			props.product.specs.forEach(spec => {
				selectedSpecs[spec.key] = matchedSku[spec.key]
			})
			return
		}
	}

	const firstAvailableSku = props.product.skulist.find(sku => sku.stock > 0)
	if (firstAvailableSku) {
		props.product.specs.forEach(spec => {
			selectedSpecs[spec.key] = firstAvailableSku[spec.key]
		})
	} else {
		props.product.specs.forEach(spec => {
			selectedSpecs[spec.key] = ''
		})
	}
}

// 监听 props.product 变化时重新初始化
watch(
	() => props.product,
	() => {
		initDefaultSelection()
	},
	{ immediate: true },
)

// 日历
const formatDate = date => {
	return `${date.getFullYear()}/${date.getMonth() + 1}/${date.getDate()}`
}
// 日期选择器确认回调
const DateonConfirm = value => {
	isShowDate.value = false
	date.value = formatDate(value)
}

// 缓存 window.innerWidth 避免重复读取
let cachedWindowWidth = null
const getCachedWindowWidth = () => {
	if (cachedWindowWidth === null) {
		cachedWindowWidth = window.innerWidth
	}
	return cachedWindowWidth
}
/**
 * 将 px 转换为当前设备的实际 px（基于设计稿宽度）
 * @param {number} px - 设计稿中的 px 值
 * @param {number} designWidth - 设计稿宽度（默认 750）
 * @returns {number} - 转换后的 px 值
 */
// 优化的 unitToPx，使用缓存的窗口宽度
const unitToPxOptimized = (px, designWidth = 750) => {
	const windowWidth = getCachedWindowWidth()
	// 简单判断用户项目是否为 375 设计稿（通过屏幕宽度）
	const isLikely375Design = windowWidth < 400
	if (isLikely375Design) {
		// 若为 375 项目，则 px 值减半（近似兼容）
		return (px / 2) * (windowWidth / 375)
	}
	// 默认 750 项目处理
	const scale = windowWidth / designWidth
	return px * scale
}

// 把 unitToPx() 结果存为 CSS 变量（优化版：批量设置减少重排）
const setCssVars = () => {
	// 使用 DocumentFragment 或批量设置来减少重排
	// 方案：一次性收集所有变量，然后批量设置
	const vars = []
	const set = (name, px) => {
		vars.push([name, `${unitToPxOptimized(px)}px`])
	}

	// 头部

	set('--popup-header-gap', 32)
	set('--popup-header-margin-bottom', 20)
	set('--popup-sku-body-max-height', 880)

	// 弹窗内边距
	set('--sku-popup-padding', 40)
	set('--sku-popup-radius', 24)

	// 图片
	set('--popup-img-size', 180)
	set('--popup-img-radius', 8)

	// 价格
	set('--popup-price-font-size', 40)

	// 库存
	set('--popup-stock-font-size', 24)
	set('--popup-stock-margin', 4)

	// 原价
	set('--origin-price-margin-left', 12)
	set('--default-origin-price-margin-left', 24)
	set('--default-origin-price-font-size', 24)

	// 已选
	set('--popup-selected-font-size', 24)

	// 文本框
	set('--popup-form-font-size', 28)

	// 规格组
	set('--spec-group-margin-bottom', 24)

	// 规格标题
	set('--spec-title-font-size', 30)
	set('--spec-title-margin-bottom', 16)

	// 规格选项
	set('--spec-options-gap', 16)

	// 规格标签
	set('--spec-tag-padding-v', 12)
	set('--spec-tag-padding-h', 24)

	// 数量组
	set('--quantity-group-gap', 20)
	set('--quantity-group-margin', 24)

	// 数量标签
	set('--quantity-label-font-size', 30)

	// 限购
	set('--quota-text-font-size', 24)

	// 按钮
	set('--action-buttons-margin-top', 40)
	set('--action-buttons-radius', 50)
	set('--action-buttons-shadow-y', 4)
	set('--action-buttons-shadow-blur', 12)

	// 图标
	set('--popup-icon-size', 44)
	set('--popup-icon-radius', 4)

	// 分期
	set('--installment-group-margin-bottom', 24)
	set('--installment-title-font-size', 30)
	set('--installment-title-margin-bottom', 16)
	set('--installment-tips-font-size', 24)
	set('--installment-tips-margin-left', 12)
	set('--installment-radio-margin-right', 24)
	set('--installment-radio-margin-bottom', 16)

	// 使用 requestAnimationFrame 批量设置，减少重排
	requestAnimationFrame(() => {
		const root = document.documentElement
		vars.forEach(([name, value]) => {
			root.style.setProperty(name, value)
		})
		// 设置完成后清除缓存，以便窗口大小变化时重新计算
		cachedWindowWidth = null
	})
}

// 页面加载时设置变量（延迟到下一个 tick，确保 DOM 已准备好）
onMounted(() => {
	// 使用 nextTick 确保在 DOM 更新后设置样式
	nextTick(() => {
		setCssVars()
	})
})
</script>

<style scoped>
.vant4-sku-popup {
	padding: var(--sku-popup-padding);
	border-radius: var(--sku-popup-radius) var(--sku-popup-radius) 0 0;
	display: flex;
	flex-direction: column;
	align-items: stretch;
	min-height: 50%;
	max-height: 80%;

	overflow-y: visible;
	background: #fff;
}

/* 使用 CSS 变量 */
.vant4-popup-header {
	display: flex;
	align-items: center;
	cursor: pointer;
	position: relative;
	gap: var(--popup-header-gap);
	margin-bottom: var(--popup-header-margin-bottom);
}
.vant4-popup-info {
	flex: 1;
	display: flex;
	flex-direction: column;
	justify-content: center;
}

.vant4-popup-img {
	width: var(--popup-img-size);
	height: var(--popup-img-size);
	object-fit: cover;
	border-radius: var(--popup-img-radius);
}

.vant4-popup-icon {
	position: absolute;
	top: 0;
	right: 0;
	z-index: 3;
	width: var(--popup-icon-size);
	height: var(--popup-icon-size);
	line-height: var(--popup-icon-size);
	color: #fff;
	text-align: center;
	background-color: rgba(0, 0, 0, 0.4);
	border-bottom-left-radius: var(--popup-icon-radius);
}

.vant4-popup-price {
	font-size: var(--popup-price-font-size);
	color: #ee0a24;
	margin: 0;
}

.vant4-popup-stock {
	color: #666;
	font-size: var(--popup-stock-font-size);
	margin: var(--popup-stock-margin) 0;
}

.vant4-default-origin-price {
	margin-left: var(--default-origin-price-margin-left);
	font-size: var(--default-origin-price-font-size);
	color: #999;
	text-decoration: line-through;
}
.vant4-slot-origin-price {
	margin-left: var(--default-origin-price-margin-left);
}

.vant4-popup-selected {
	color: #666;
	font-size: var(--popup-selected-font-size);
}

.vant4-sku-body {
	flex: 1 1 auto;
	overflow-y: scroll;
	max-height: var(--popup-sku-body-max-height);
	-ms-overflow-style: none; /* IE/Edge */
	scrollbar-width: none; /* Firefox */
}
/* 隐藏滚动条但保持滚动功能（兼容性写法） */
.vant4-sku-body::-webkit-scrollbar {
	width: 0;
	height: 0;
}

.vant4-sku-body ::v-deep(.van-cell__title) {
	font-size: var(--quantity-label-font-size) !important;
	color: #333 !important;
}

.vant4-sku-body ::v-deep(.van-cell__value) {
	font-size: var(--popup-form-font-size) !important;
	color: #666 !important;
}

.vant4-sku-body ::v-deep(.van-field) {
	font-size: var(--popup-form-font-size) !important;
	color: #666 !important;
	padding-left: 0 !important;
	padding-right: 0 !important;
}

.vant4-sku-body ::v-deep(.van-cell) {
	font-size: var(--popup-form-font-size) !important;
	color: #666 !important;
	padding-left: 0 !important;
	padding-right: 0 !important;
}
.vant4-sku-body ::v-deep(.van-cell-group) {
	padding-left: 0 !important;
	padding-right: 0 !important;
}

.vant4-spec-group {
	margin-bottom: var(--spec-group-margin-bottom);
}

.vant4-spec-title {
	font-size: var(--spec-title-font-size);
	font-weight: bold;
	margin-bottom: var(--spec-title-margin-bottom);
	color: #333;
}

.vant4-spec-options {
	display: flex;
	flex-wrap: wrap;
	gap: var(--spec-options-gap);
}

.vant4-spec-tag {
	padding: var(--spec-tag-padding-v) var(--spec-tag-padding-h);
	cursor: pointer;
}

.vant4-quantity-group {
	display: flex;
	align-items: center;
	justify-content: space-between;
	gap: var(--quantity-group-gap);
	margin: var(--quantity-group-margin) 0;
}

.vant4-quantity-label {
	font-size: var(--quantity-label-font-size);
	font-weight: bold;

	color: #333;
}
.vant4-quota-group {
	display: flex;
	justify-content: flex-end;
	align-items: center;
	flex: 1;
}
.vant4-quota-text {
	float: right;
	color: #ee0a24;
	font-size: var(--quota-text-font-size);
}

.vant4-action-buttons {
	display: flex;
	width: 100%;
	margin-top: var(--action-buttons-margin-top);
	border-radius: var(--action-buttons-radius);
	overflow: hidden;
	box-shadow: 0 var(--action-buttons-shadow-y) var(--action-buttons-shadow-blur) rgba(0, 0, 0, 0.15);
	position: relative;
}

.vant4-installment-group {
	margin-bottom: var(--installment-group-margin-bottom);
}
.vant4-installment-title {
	font-size: var(--installment-title-font-size);
	font-weight: bold;
	color: #333;
	margin-bottom: var(--installment-title-margin-bottom);
}

.vant4-installment-tips {
	font-size: var(--installment-tips-font-size);
	color: #ee0a24;
	margin-left: var(--installment-tips-margin-left);
}

.vant4-installment-radio {
	margin-right: var(--installment-radio-margin-right);
	margin-bottom: var(--installment-radio-margin-bottom);
}
</style>
