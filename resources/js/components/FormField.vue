<template>
    <field-wrapper>
        <div class="w-1/5 px-8 py-6">
            <slot>
                <form-label
                        :label-for="field.attribute"
                        :class="{ 'mb-2': showHelpText && field.helpText }"
                >
                    {{ fieldLabel }}

                    <span v-if="field.required" class="text-danger text-sm">{{
                        __('*')
                    }}</span>
                </form-label>
            </slot>
        </div>
        <div class="px-8 py-6" :class="computedWidth">
            <a
                class="inline-block font-bold cursor-pointer mr-2 animate-text-color select-none border-primary"
                :class="{ 'text-60': localeKey !== currentLocale, 'text-primary border-b-2': localeKey === currentLocale }"
                :key="`a-${localeKey}`"
                v-for="(locale, localeKey) in field.locales"
                @click="changeTab(localeKey)"
            >
                {{ locale }}
            </a>

            <textarea
                ref="field"
                :id="field.name"
                class="mt-4 w-full form-control form-input form-input-bordered py-3 min-h-textarea"
                :class="errorClasses"
                :placeholder="field.name"
                :disabled="isReadonly"
                v-model="value[currentLocale]"
                v-if="!field.singleLine && !field.trix && !field.markdown"
                @keydown.tab="handleTab"
            ></textarea>

            <div v-if="!field.singleField && field.trix" @keydown.stop class="mt-4">
                <trix
                    ref="field"
                    name="trixman"
                    :value="value[currentLocale]"
                    placeholder=""
                    @change="handleChange"
                />
            </div>

            <div v-if="!field.singleField && field.markdown" @keydown.stop class="mt-4">
                <textarea
                    ref="MarkdownArea"
                    class="w-full form-control form-input form-input-bordered"

                    :id="field.name"
                    :class="errorClasses"
                    :placeholder="field.name"
                    :value="value[currentLocale]"
                />
            </div>

            <input
                ref="field"
                type="text"
                :id="field.name"
                class="mt-4 w-full form-control form-input form-input-bordered"
                :class="errorClasses"
                :placeholder="field.name"
                :disabled="isReadonly"
                v-model="value[currentLocale]"
                v-if="field.singleLine"
                @keydown.tab="handleTab"
            />

            <p v-if="hasError" class="my-2 text-danger">
                {{ firstError }}
            </p>
            <help-text class="help-text mt-2" v-if="field.helpText">
                {{ field.helpText }}
            </help-text>
        </div>
    </field-wrapper>
</template>

<script>

import Trix from '../Trix'
import easyMDE from "easymde";
import { FormField, HandlesValidationErrors } from 'laravel-nova'

require("easymde/dist/easymde.min.css");

export default {
    mixins: [FormField, HandlesValidationErrors],

    props: ['resourceName', 'resourceId', 'field'],

    components: { Trix },

    data() {
        return {
            locales: Object.keys(this.field.locales),
            currentLocale: this.field.currentLocale,
        }
    },

    mounted() {
        if (!this.field.singleField && this.field.markdown) {
            this.$refs.MarkdownArea.value =(this.field.value || {})[this.field.currentLocale] || "";
            this.easymde = new easyMDE({
                element: this.$refs.MarkdownArea,
                spellChecker: false,
                hideIcons: ["image"],
                showIcons: ["table"],
            });
            this.easymde.codemirror.on("change", (cm, changeObj) => {
                this.value[this.currentLocale] = this.easymde.value();
            });
        }
    },

    methods: {
        /*
         * Set the initial, internal value for the field.
         */
        setInitialValue() {
          this.value = this.field.value || {}
        },

        /**
         * Fill the given FormData object with the field's internal value.
         */
        fill(formData) {
            Object.keys(this.value).forEach(locale => {
                formData.append(this.field.attribute + '[' + locale + ']', this.value[locale] || '')
            })
        },

        /**
         * Update the field's internal value.
         */
        handleChange(value) {
          this.value[this.currentLocale] = value
        },

        changeTab(locale) {
            this.currentLocale = locale
            this.$nextTick(() => {
                if (this.field.trix) {
                    this.$refs.field.update()
                } else if (this.field.markdown) {
                    this.easymde.value((this.field.value || {})[this.currentLocale] || "");
                } else {
                    this.$refs.field.focus()
                }
            })
        },

        handleTab(e) {
            const currentIndex = this.locales.indexOf(this.currentLocale)
            if (!e.shiftKey) {
                if (currentIndex < this.locales.length - 1) {
                    e.preventDefault()
                    this.changeTab(this.locales[currentIndex + 1])
                }
            } else {
                if (currentIndex > 0) {
                    e.preventDefault()
                    this.changeTab(this.locales[currentIndex - 1])
                }
            }
        }
    },

    computed: {
        computedWidth() {
            return {
                'w-1/2': !this.field.trix,
                'w-4/5': this.field.trix
            }
        },

        /**
        * Return the label that should be used for the field.
        */
        fieldLabel() {
        // If the field name is purposefully an empty string, then let's show it as such
            if (this.fieldName === '') {
              return ''
            }

            return this.fieldName || this.field.singularLabel || this.field.name
        },
    }
}
</script>
