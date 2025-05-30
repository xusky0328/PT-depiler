<script setup lang="ts">
import { ref } from "vue";
import { useI18n } from "vue-i18n";
import { computedAsync } from "@vueuse/core";
import { getMediaServer, getMediaServerMetaData, IMediaServerMetadata } from "@ptd/mediaServer";

import type { IMediaServerMetadata as IMediaServerConfig } from "@/shared/types.ts";
import { formatDate, formValidateRules } from "@/options/utils.ts";

import ConnectCheckButton from "@/options/components/ConnectCheckButton.vue";

const { t } = useI18n();

const clientConfig = defineModel<IMediaServerConfig>();
const emits = defineEmits<{
  (e: "update:configValid", value: boolean): void;
}>();

const clientMeta = computedAsync<IMediaServerMetadata>(
  async () => await getMediaServerMetaData(clientConfig.value!.type),
  {} as IMediaServerMetadata,
);
const formValid = ref<boolean>(false);

async function checkConnect() {
  if (formValid) {
    const client = await getMediaServer(clientConfig.value!);
    return await client.ping();
  }
  return false;
}
</script>

<template>
  <v-card class="mb-5">
    <v-form v-model="formValid" v-if="clientConfig" fast-fail>
      <v-container class="pa-0">
        <v-row>
          <v-col cols="12" md="4">
            <v-text-field v-model="clientConfig.type" :label="t('SetDownloader.common.type')" disabled />
          </v-col>
          <v-col cols="12" md="4">
            <v-text-field
              v-model="clientConfig.name"
              :label="t('SetDownloader.common.name')"
              :placeholder="t('SetDownloader.common.name')"
              :rules="[formValidateRules.require(t('SetDownloader.editor.nameTip'))]"
              required
            />
          </v-col>
          <v-col cols="12" md="4">
            <v-text-field
              v-model="clientConfig.id"
              :label="t('SetDownloader.common.uid') + t('SetDownloader.editor.uidPlaceholder')"
              disabled
            />
          </v-col>
        </v-row>
        <v-row>
          <v-text-field
            v-model="clientConfig.address"
            :label="t('SetDownloader.common.address')"
            :rules="[formValidateRules.url(t('SetDownloader.editor.addressTip'))]"
            required
          />
        </v-row>

        <v-row>
          <v-col class="py-0"><v-label>认证信息</v-label></v-col>
          <v-col v-for="(auth_field, index) in clientMeta!.auth_field" :key="index" cols="12">
            <template v-if="typeof auth_field === 'string'">
              <v-text-field
                v-model="clientConfig.auth[auth_field]"
                :label="auth_field"
                :rules="[formValidateRules.require()]"
                hide-details
                required
              />
            </template>
            <template v-else>
              <v-text-field
                v-model="clientConfig.auth[auth_field.name]"
                :label="auth_field.name"
                :messages="auth_field.message ?? ''"
                :rules="auth_field.required ? [formValidateRules.require()] : []"
                required
              />
            </template>
          </v-col>
        </v-row>

        <v-row>
          <v-slider
            v-model="clientConfig.timeout"
            :color="clientConfig.timeout! > 8 * 60e3 ? 'red' : clientConfig.timeout! > 5 * 60e3 ? 'amber' : 'green'"
            :label="t('SetDownloader.editor.timeout')"
            :max="10 * 60e3"
            :min="0"
            :step="1e3"
            class="px-2"
          >
            <template #append>
              <v-btn variant="flat" @click="clientConfig.timeout = 60e3">
                {{ formatDate(clientConfig.timeout!, "mm:ss") }}
              </v-btn>
            </template>
          </v-slider>
        </v-row>

        <ConnectCheckButton
          :check-fn="checkConnect"
          :reset-timeout="3e3"
          @after:check-connect="
            () => emits('update:configValid', formValid && true) // 不管是否测试成功，都允许用户进行下一步操作（保存下载服务器配置）
          "
        />

        <v-alert v-if="clientMeta?.warning" color="warning">
          <ul>
            <li v-for="(data, index) in clientMeta.warning" :key="index">● {{ data }}</li>
          </ul>
        </v-alert>
      </v-container>
    </v-form>
  </v-card>
</template>

<style scoped lang="scss"></style>
