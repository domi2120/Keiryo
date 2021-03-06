<!-- This Source Code Form is subject to the terms of the Mozilla Public
   - License, v. 2.0. If a copy of the MPL was not distributed with this
   - file, You can obtain one at https://mozilla.org/MPL/2.0/. -->

<template>
    <div class="field select-component">
        <ValidationProvider
            tag="div"
            :rules="rules"
            :vid="name"
            v-slot="{ errors }">
            <div class="field-container" :class="{ 'errored': errors.length > 0 }">
                <label class="label" v-if="label">{{ label }}</label>
                <select
                    @change="onSelection()"
                    @search="$emit('search', $event)"
                    v-model="selected"
                    ref="selectElement"
                />
            </div>
        </ValidationProvider>
    </div>
</template>

<script lang="ts">
import { Component, Prop, Vue, Watch } from "vue-property-decorator";
import SelectListItem from "../models/SelectListItem";
import Choices from "choices.js";
import _ from "lodash";

@Component
export default class SelectComponent extends Vue {
    @Prop({ required: true }) private name: string;
    @Prop({ default: "" }) private label: string;
    @Prop({ default: "" }) private rules: string;

    @Prop({ default: true }) private searchEnabled: boolean;
    @Prop({ default: false }) protected multiple: boolean;
    @Prop({ default: true }) private placeholder: boolean;
    @Prop({ required: true }) private elements: SelectListItem[];

    private selected: string | string[] = [];
    private dropdown: Choices;

    mounted() {
        const element: HTMLSelectElement = this.$refs.selectElement as HTMLSelectElement;
        element.multiple = this.multiple;

        if (element) {
            let choices: SelectListItem[] = this.prepareElements();

            this.dropdown = new Choices(element, {
                searchEnabled: this.searchEnabled,
                choices: choices,
                items: [],
                itemSelectText: "",
                loadingText: "",
                removeItemButton: this.multiple
            });
        }

        // When elements are present on creation then map the selection and add it to the v-model.
        if (this.elements.length > 0) {
            this.mapElementSelectionToSelection();
            this.$emit("input", this.selected);
        }
    }

    @Watch("elements") onElementChange() {
        this.mapElementSelectionToSelection();
        this.updateChoices(this.updateElementSelection(this.elements));
    }

    private mapElementSelectionToSelection() {
        const selectedItems = this.elements.filter(x => x.selected && x.value);

        if (!this.multiple) {
            this.selected = _.first(selectedItems)?.value;
        } else {
            this.selected = selectedItems.map(x => x.value);
        }
    }

    private onSelection() {
        this.$nextTick(() => {
            if (!this.multiple) {
                const updatedElements = this.updateElementSelection(this.elements);
                this.updateChoices(updatedElements);
            }

            this.$emit("input", this.selected);
            this.$emit("selection", this.selected);
        });
    }

    private updateElementSelection(elements: SelectListItem[]): SelectListItem[] {
        let updatedElements: SelectListItem[] = [];

        elements.forEach(x => {
            if (x.value == this.selected) {
                x.selected = true;
                updatedElements.push(x);
            } else {
                x.selected = false;
                updatedElements.push(x);
            }
        });

        return updatedElements;
    }

    private updateChoices(elements: SelectListItem[]) {
        const newElements = this.prepareElements(elements);

        this.dropdown.clearChoices();
        this.dropdown.setChoices(newElements);
    }

    private prepareElements(elements: SelectListItem[] = this.elements): SelectListItem[] {
        let choices = elements;

        // Add placeholder when there is no selected item.
        if (this.placeholder && !this.multiple) {
            if (!choices.some(x => x.selected)) {
                choices.unshift({
                    label: "",
                    value: null,
                    selected: true,
                    disabled: true
                });

                if (this.dropdown) {
                    this.dropdown.setValue([""]);
                }
            }
        }

        return choices;
    }
}
</script>

<style lang="scss">
@use "../colors.scss";

.select-component {
    .errored {
        .choices__inner {
            border-color: colors.$color-attention;
        }
    }
}
</style>
