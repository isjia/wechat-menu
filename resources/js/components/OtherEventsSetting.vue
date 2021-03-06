<template>
    <div class="wm-other-events-setting">
        <table>
            <tr>
                <th
                    v-for="(c, index) of columns"
                    :key="index"
                    :style="{ width: c.width}"
                >{{ c.name }}
                </th>
            </tr>
            <tr
                v-for="(e, index) of events"
                :key="index"
            >

                <td
                    v-if="isDefaultEvent(e)"
                    :style="{ width: columns[0].width }"
                >
                    {{ typeText(e.key) }}({{ e.key }})
                </td>
                <td
                    v-else
                    :style="{ width: columns[0].width }"
                >
                    <w-input
                        :has-error="hasError('key', index)"
                        :error-text="getError('key', index)"
                        v-model="e.key"
                        error-inside
                    />
                </td>

                <td
                    :style="{ width: columns[1].width }"
                >
                    <change-handle-type :event="e"/>
                </td>
                <td
                    class="wm-table-content"
                    :style="{ width: columns[2].width }"
                >
                    <media-picker
                        v-if="e.type == 'msg'"
                        v-model="e.content"
                        :has-error="hasError('content', index)"
                        :error-text="getError('content', index)"
                        error-inside
                        ref="inputs"
                    />
                    <w-callback-input
                        v-else
                        v-model="e.content"
                        :events="events"
                        group="otherEvents"
                        ref="inputs"
                        :has-error="hasError('content', index)"
                        :error-text="getError('content', index)"
                        error-inside
                    />
                </td>
                <td
                    :style="{ width: columns[3].width }"
                >
                    <button
                        v-if="!isDefaultEvent(e)"
                        class="wm-btn wm-btn-danger wm-btn-sm"
                        @click="onRemove(index)"
                    >删除</button>
                </td>
            </tr>
        </table>
        <div class="wm-footer-toolbar">
            <button
                class="wm-btn wm-btn-primary"
                @click="onSave($v, events, 'other_events')"
                :disabled="saving"
            >保存</button>
            <refresh :on-refresh="getData"/>
            <button
                class="wm-btn"
                @click="onReset"
            >重置</button>
            <button
                class="wm-btn"
                @click="onNewEvent"
            >添加一个</button>
        </div>
    </div>
</template>

<script>
import ChangeHandleType from '@/components/ChangeHandleType'
import CallbackInput from '@/components/CallbackInput'
import { OTHER_EVENT_TYPES } from '@/common/constants'
import { getResources } from '@/api/wechat'
import { required } from 'vuelidate/lib/validators'
import { callback } from '@/common/validators'
import eventErrorHelper from '@/common/event-error-helper'
import createEvents from '@/common/create-events'
import MediaPicker from '@/components/media-picker/MediaPicker'
import { setHeadNoReferrer } from '@/common/utils'

export default {
    name: 'OtherEventsSetting',
    components: {
        ChangeHandleType,
        CallbackInput,
        MediaPicker,
    },
    mixins: [
        eventErrorHelper,
        createEvents,
    ],
    data() {
        return {
            columns: [
                {
                    field: 'name',
                    name: '事件',
                    width: '200px',
                },
                {
                    field: 'type',
                    name: '处理方法',
                    width: '100px',
                },
                {
                    field: 'content',
                    name: '内容',
                },
                {
                    field: 'actions',
                    name: '操作',
                    width: '80px',
                },
            ],
            events: [],
            saving: false,

            fieldErrors: {
                key: {
                    required: '必须填写',
                },
                content: {
                    required: '必须设置',
                    callback: '不是有效的回调',
                },
            },
        }
    },
    validations: {
        events: {
            $each: {
                key: {
                    required,
                },
                content: {
                    // required,
                    callback,
                },
            },
        },
    },
    created() {
        setHeadNoReferrer()
        this.getData()
    },
    methods: {
        typeText(type) {
            return OTHER_EVENT_TYPES[type] || type
        },
        async getData() {
            const { data } = await getResources('other_events')
            this.events = data.data
            this.initDefaultEvents()
        },
        /**
         * 初始化事件设定，比如首次使用时，一些默认的事件用户是没有配置的。或者之后用户删除了某些默认事件
         */
        initDefaultEvents() {
            // 预设事件，就是 OTHER_EVENT_TYPES 常量中的
            const defaultEvents = []
            // 用户添加的其他事件，可能是我漏掉没有放到预设中的，或者公众号后期新增的
            const userEvents = []
            // key 与 事件配置 的键值对
            const mappedEvents = {}

            this.events.forEach(e => {
                if (OTHER_EVENT_TYPES[e.key]) {
                    mappedEvents[e.key] = e
                } else {
                    userEvents.push(e)
                }
            })

            // 如果请求获取的事件配置数据中，没有某些预设事件的配置，则给个默认的
            for (let key of Object.keys(OTHER_EVENT_TYPES)) {
                if (!mappedEvents[key]) {
                    defaultEvents.push({
                        key,
                        type: 'callback',
                        content: '',
                    })
                } else {
                    defaultEvents.push(mappedEvents[key])
                }
            }

            // 把预设的放前面，用户自行添加的全部放后面
            this.events = [...defaultEvents, ...userEvents]
            this.eventsBak = JSON.stringify(this.events)
        },
        onNewEvent() {
            this.events.push({
                key: '',
                type: 'callback',
                content: '',
            })

            this.$nextTick(() => {
                this.$refs.inputs[this.events.length - 1].focus()
            })
        },
        onReset() {
            this.events = JSON.parse(this.eventsBak)
        },
        isDefaultEvent(event) {
            return !!OTHER_EVENT_TYPES[event.key]
        },
        onRemove(index) {
            const e = this.events[index]
            if (this.isDefaultEvent(e)) {
                this.$notice({
                    msg: '不能删除预设的事件',
                    type: 'error',
                })
                return
            }

            confirm('确定删除？') && this.events.splice(index, 1)
        },
    },
}
</script>
