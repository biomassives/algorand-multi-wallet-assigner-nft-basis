<script setup lang="ts">
import { computed, onMounted, ref } from "vue";
import { useStore } from "vuex";
import Dropdown from "primevue/dropdown";
import { useI18n } from "vue-i18n";

const props = defineProps({
  modelValue: String,
  itemId: String,
});

const emit = defineEmits(["update:modelValue"]);

const model = computed({
  get() {
    return props.modelValue;
  },

  set(value) {
    return emit("update:modelValue", value);
  },
});

const store = useStore();
const { t } = useI18n();

const options = await store.dispatch("publicData/getTwoFactorAuthList", {
  chainId: store.state.config.env,
});

if (options && options.length == 1) {
  model.value = options[0].id;
}
console.log("options", options);
</script>
<template>
  <Dropdown
    v-model="model"
    filter
    :options="options"
    optionLabel="providerName"
    optionValue="id"
    :placeholder="$t('account.select_account')"
    class="w-full md:w-14rem"
    :itemid="props.itemId"
  >
    <template #option="slotProps">
      <div v-if="slotProps.option" class="flex align-items-center">
        <div>2FA service by {{ slotProps.option.providerName }}</div>
      </div>
    </template>
  </Dropdown>
</template>
