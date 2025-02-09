<script setup lang="ts">
import { mapActions, useStore } from "vuex";
import MainLayout from "../../layouts/Main.vue";
import { reactive, onMounted } from "vue";
import { useI18n } from "vue-i18n";
import InputMask from "primevue/inputmask";
import Password from "primevue/password";
import Button from "primevue/button";
import SelectAccount from "@/components/SelectAccount.vue";
import Select2FAServer from "@/components/Select2FAServer.vue";
import { wordlist } from "@scure/bip39/wordlists/english";
import sha512 from "js-sha512";
import { Shamir } from "@spliterati/shamir";
import algosdk from "algosdk";

import { useRouter } from "vue-router";
const store = useStore();
const router = useRouter();
const { t } = useI18n(); // use as global scope
const state = reactive({
  mn: [""],
  name: "",
});
// from Uint11Array
// https://github.com/algorand/js-algorand-sdk/blob/7965d1c194186e5c7b8a86756c546f2ec35291cd/src/mnemonic/mnemonic.ts#L69
function toUint8Array(buffer11: number[]) {
  const buffer8: Array<number> = [];
  let acc = 0;
  let accBits = 0;
  function add(ui11: number) {
    acc |= ui11 << accBits;
    accBits += 11;
    while (accBits >= 8) {
      buffer8.push(acc & 0xff);
      acc >>= 8;
      accBits -= 8;
    }
  }
  function flush() {
    if (accBits) {
      buffer8.push(acc);
    }
  }

  buffer11.forEach(add);
  flush();
  return new Uint8Array(buffer8);
}

function seedFromMnemonic(mnemonic: string) {
  const words = mnemonic.split(" ");
  const key = words.slice(0, words.length - 1);

  // Check that all words are in list
  for (const w of words) {
    if (wordlist.indexOf(w) === -1)
      throw new Error("NOT_IN_WORDS_LIST_ERROR_MSG");
  }

  const checksum = words[words.length - 1];
  const uint11Array = key.map((word) => wordlist.indexOf(word));

  // Convert the key to uint8Array
  let uint8Array = toUint8Array(uint11Array);
  const data = uint8Array.slice(0, 33);
  console.log("uint8Array", key, uint8Array, data);
  // We dont need to chop the last byte - the shamir adds one byte extra to the array

  // compute checksum
  const cs = computeChecksum(data);

  // success!
  if (cs === checksum) return data;
  console.error("cs != checksum", cs, checksum);
  throw new Error("FAIL_TO_DECODE_MNEMONIC_ERROR_MSG");
}

function toUint11Array(buffer8: Uint8Array | number[]) {
  //https://github.com/algorand/js-algorand-sdk/blob/7965d1c194186e5c7b8a86756c546f2ec35291cd/src/mnemonic/mnemonic.ts#L12C1-L34C2
  const buffer11: Array<number> = [];
  let acc = 0;
  let accBits = 0;
  function add(octet: number) {
    acc |= octet << accBits;
    accBits += 8;
    if (accBits >= 11) {
      buffer11.push(acc & 0x7ff);
      acc >>= 11;
      accBits -= 11;
    }
  }
  function flush() {
    if (accBits) {
      buffer11.push(acc);
    }
  }

  buffer8.forEach(add);
  flush();
  return buffer11;
}
function genericHash(arr: sha512.Message) {
  return sha512.sha512_256.array(arr);
}
function applyWords(nums: number[]) {
  return nums.map((n) => wordlist[n]);
}
function computeChecksum(seed: Uint8Array) {
  const hashBuffer = genericHash(seed);
  const uint11Hash = toUint11Array(hashBuffer);
  const words = applyWords(uint11Hash);

  return words[0];
}

const recover = async () => {
  try {
    await store.dispatch("wallet/prolong");

    const bytesArr = state.mn
      .filter((mn) => !!mn)
      .map((mn) => seedFromMnemonic(mn));
    console.log("bytesArr", bytesArr);
    const reassembled = Shamir.combine(bytesArr);
    console.log("reassembled", reassembled);
    const mn = algosdk.mnemonicFromSeed(reassembled);
    console.log("mn", mn);

    await store.dispatch("wallet/addPrivateAccount", {
      mn: mn,
      name: state.name,
    });
    router.push({ name: "Accounts" });
  } catch (err: any) {
    const error = err?.message ?? err;
    await store.dispatch("toast/openError", error);
  }
};
onMounted(async () => {
  await store.dispatch("wallet/prolong");
});
</script>
<template>
  <MainLayout>
    <h1>{{ $t("new_account_shamir.header") }}</h1>
    <p>
      {{ $t("new_account_shamir.help") }}
    </p>
    <div v-for="(mn, index) in state.mn">
      <label :for="`mn${index}`"
        >{{ $t("new_account_shamir.mnemonic") }} #{{ index + 1 }}</label
      >
      <Password
        v-model="state.mn[index]"
        :feedback="false"
        :inputId="`mn${index}`"
        class="m-2 w-100"
        inputClass="w-100"
        inn
        toggleMask
      ></Password>
    </div>
    <Button @click="state.mn.push('')" class="m-2 w-100" severity="secondary">{{
      $t("new_account_shamir.add_mnemnic")
    }}</Button>
    <div>
      <label for="name">{{ $t("new_account_shamir.account_name") }}</label>
      <InputText
        v-model="state.name"
        inputId="name"
        class="m-2 w-100"
      ></InputText>
    </div>
    <Button @click="recover" class="m-2 w-100" severity="primary">{{
      $t("new_account_shamir.recover")
    }}</Button>
  </MainLayout>
</template>
