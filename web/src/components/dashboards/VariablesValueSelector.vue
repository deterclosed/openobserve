<!-- Copyright 2022 Zinc Labs Inc. and Contributors
 Licensed under the Apache License, Version 2.0 (the "License");
 you may not use this file except in compliance with the License.
 You may obtain a copy of the License at
     http:www.apache.org/licenses/LICENSE-2.0
 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License. 
-->
<template>
  <div v-if="variablesData.values?.length > 0 && !variablesData.isVariablesLoading" class="flex q-mt-sm q-ml-sm">
    <div v-for="item in variablesData.values" class="q-mr-lg q-mt-sm">
      <div v-if="item.type == 'query_values'">
        <q-select style="min-width: 150px;"  filled outlined dense v-model="item.value" :options="item.options"
          :label="item.label || item.name"></q-select>
      </div>
      <div v-else-if="item.type == 'constant'">
        <q-input style="max-width: 150px !important" v-model="item.value" :label="item.label || item.name" dense outlined
          readonly></q-input>
      </div>
      <div v-else-if="item.type == 'textbox'">
        <q-input style="max-width: 150px !important" debounce="500" v-model="item.value" :label="item.label || item.name" dense
          outlined></q-input>
      </div>
      <div v-else-if="item.type == 'custom'">
        <q-select style="min-width: 150px;" outlined dense v-model="item.value" :options="item.options" map-options filled borderless
          :label="item.label || item.name" option-value="value" option-label="label" emit-value></q-select>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { onMounted, watch } from 'vue';
import { defineComponent, reactive } from 'vue';
import streamService from "../../services/stream";
import { useStore } from 'vuex';

export default defineComponent({
  name: "VariablesValueSelector",
  props: ["selectedTimeDate", "variablesConfig"],
  emits: ["variablesData"],
  setup(props: any, { emit }) {
    const store = useStore()

    // variables data derived from the variables config list 
    const variablesData: any = reactive({
      isVariablesLoading: false,
      values: []
    })

    onMounted(() => {
      getVariablesData()
    })

    // you may need to query the data if the variable configs or the data/time changes
    watch(() => [props.variablesConfig, props.selectedTimeDate], () => {
      getVariablesData()
    })

    watch(() => variablesData, () => {
      emit('variablesData', variablesData)
    })

    const getVariablesData = async () => {
      // do we have variables & date?
      if (!props.variablesConfig?.list || !props.selectedTimeDate?.start_time) {
        variablesData.values = []
        variablesData.isVariablesLoading = false
        emit('variablesData', variablesData)
        return
      }

      // get old variable values
      const oldVariableValue = JSON.parse(JSON.stringify(variablesData.values))

      // continue as we have variables
      const promise = props.variablesConfig?.list?.map((it: any) => {
        const obj: any = { name: it.name, label: it.label, type: it.type, value: "", isLoading: false }
        switch (it.type) {

          case "query_values": {
            obj.isLoading = true
            return streamService
              .fieldValues({
                org_identifier: store.state.selectedOrganization.identifier,
                stream_name: it.query_data.stream,
                start_time: new Date(props.selectedTimeDate?.start_time?.toISOString()).getTime() * 1000,
                end_time: new Date(props.selectedTimeDate?.end_time?.toISOString()).getTime() * 1000,
                fields: [it.query_data.field],
                size: 10,
                type: it.query_data.stream_type,
              })
              .then((res: any) => {
                obj.isLoading = false
                if (res.data.hits.length) {
                  //set options value from the api response
                  obj.options = res.data.hits
                    .find((field: any) => field.field === it.query_data.field)
                    .values.map((value: any) => value.zo_sql_key ? value.zo_sql_key : "null")

                    // find old value is exists in the dropdown
                  let oldVariableObjectSelectedValue = oldVariableValue.find((it2: any) => it2.name === it.name)

                  // if the old value exist in dropdown set the old value otherwise set first value of drop down otherwise set blank string value
                  if (oldVariableObjectSelectedValue) {
                    obj.value = obj.options.includes(oldVariableObjectSelectedValue.value) ? oldVariableObjectSelectedValue.value : obj.options.length ? obj.options[0] : ""
                  }
                  else {
                    obj.value = obj.options[0] || ""
                  }
                  return obj
                } else {
                  return obj
                }
              })
              .catch((err: any) => {
                return obj
              })
          }
          case "constant": {
            obj.value = it.value
            return obj
          }
          case "textbox": {
            obj.value = it.value
            return obj
          }
          case "custom": {
            obj["options"] = it.options
            let oldVariableObjectSelectedValue = oldVariableValue.find((it2: any) => it2.name === it.name)

            // if the old value exist in dropdown set the old value otherwise set first value of drop down otherwise set blank string value
            if (oldVariableObjectSelectedValue) {
              obj.value = oldVariableObjectSelectedValue.value
            }
            else {
              obj.value = obj.options[0].value || ""
            }
            return obj
            // break;
          }
          default:
            obj.value = it.value
            return obj
        }
      })
      variablesData.values = await Promise.all(promise || [])
      variablesData.isVariablesLoading = false
      emit('variablesData', variablesData)
    }

    return {
      props,
      variablesData
    };
  },
});
</script>
 