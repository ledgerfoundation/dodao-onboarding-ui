<script setup lang="ts">
import Block from '@/components/Block.vue';
import GuideViewStepper from '@/components/Guide/View/Stepper.vue';
import Icon from '@/components/Icon.vue';
import LayoutSingle from '@/components/Layout/Single.vue';
import ModalConfirm from '@/components/Modal/Confirm.vue';
import ModalGuideGuideExport from '@/components/Modal/Guide/GuideExport.vue';
import PageLoading from '@/components/PageLoading.vue';
import UiDropdown from '@/components/Ui/Dropdown.vue';
import UiMarkdown from '@/components/Ui/Markdown.vue';
import User from '@/components/User.vue';
import { useViewGuide } from '@/composables/guide/useViewGuide';
import { useApp } from '@/composables/useApp';
import { useClient } from '@/composables/useClient';
import { useProfiles } from '@/composables/useProfiles';
import { useSharing } from '@/composables/useSharing';
import { useSpace } from '@/composables/useSpace';
import { useStore } from '@/composables/useStore';
import { useWeb3 } from '@/composables/useWeb3';
import { getIpfsUrl, setPageTitle } from '@/helpers/utils';
import { SpaceModel } from '@dodao/onboarding-schemas/models/SpaceModel';
import { computed, inject, onMounted, PropType, ref, watch } from 'vue';
import { useI18n } from 'vue-i18n';
import { useRoute, useRouter } from 'vue-router';

const props = defineProps({
  spaceId: String,
  space: { type: Object as PropType<SpaceModel>, required: true },
  spaceLoading: Boolean,
  from: { type: String }
});

const { isAdmin, isSuperAdmin } = useSpace(props.space);

const route = useRoute();
const router = useRouter();
const { t } = useI18n();
const { isEthBlockchain, isOneBlockchain, web3, web3Account } = useWeb3();
const { send, clientLoading } = useClient();
const { getExplore } = useApp();
const { store } = useStore();
const notify = inject('notify') as Function;

const uuid = route.params.uuid;

const modalOpen = ref(false);

const modalGuideExportOpen = ref(false);

const backButtonText = props.from
  ? JSON.parse(props.from)?.displayName || props.space.name
  : props.space.name;

const {
  activeStepId,
  goToNextStep,
  goToPreviousStep,
  guideRef: guide,
  guideLoaded,
  guideResponseRef,
  guideSubmittingRef,
  initialize,
  selectAnswer,
  submitGuide,
  setUserInput
} = useViewGuide(uuid as string, notify, props.space);

const loaded = computed(() => !props.spaceLoading && guideLoaded.value);

const threeDotItems = computed(() => {
  const items: Array<{ text: string; action: string }> = [];

  // items.push({ text: t('guide.duplicate'), action: 'duplicate' });
  // items.push({ text: t('guide.delete'), action: 'delete' });
  items.push({ text: t('guide.edit'), action: 'edit' });
  items.push({ text: t('guide.submissions'), action: 'viewSubmissions' });

  if (isSuperAdmin) {
    items.push({ text: t('guide.export'), action: 'exportGuide' });
  }
  return items;
});

async function deleteGuide() {
  const result = await send(props.space, 'delete-guide', {
    guide: guide.value
  });
  console.log('Result', result);
  if (result?.id) {
    await getExplore();
    store.space.guides = [];
    notify?.(['green', t('notify.guideDeleted')]);
    await router.push({ name: 'guide' });
  }
}

async function editGuide() {
  await router.push({
    name: 'guideEdit',
    params: { uuid: guide.value?.uuid, settingTab: 'basic' }
  });
}

async function viewSubmissions() {
  await router.push({
    name: 'guideSubmissions',
    params: { uuid: guide.value?.uuid }
  });
}

async function exportGuide() {
  modalGuideExportOpen.value = true;
}

const {
  shareToTwitter,
  shareToFacebook,
  shareToClipboard,
  startShare,
  sharingIsSupported,
  sharingItems
} = useSharing();

function selectFromThreedotDropdown(e) {
  if (e === 'delete') deleteGuide();
  if (e === 'edit') editGuide();
  if (e === 'exportGuide') exportGuide();
  if (e === 'viewSubmissions') viewSubmissions();
  if (e === 'duplicate')
    router.push({
      name: 'spaceCreate',
      params: {
        key: props.spaceId,
        from: guide.value?.id
      }
    });
}

function selectFromShareDropdown(e) {
  if (e === 'shareToTwitter')
    shareToTwitter(
      props.space,
      guide.value?.name ?? '',
      'guide/view',
      guide.value?.uuid ?? '',
      window
    );
  else if (e === 'shareToFacebook')
    shareToFacebook(
      props.space,
      guide.value?.name ?? '',
      'guide/view',
      guide.value?.uuid ?? '',
      window
    );
  else if (e === 'shareToClipboard')
    shareToClipboard(props.space, 'guide/view', guide.value?.uuid ?? '');
}

const { profiles, loadProfiles } = useProfiles();

watch(guide, () => {
  guide.value?.authors && loadProfiles([...guide.value.authors]);
});

watch([loaded, web3Account], () => {
  if (web3.value.authLoading && !web3Account.value) return;
  if (!loaded.value) return;
});

onMounted(async () => {
  await initialize();
  setPageTitle('page.title.dao.guide', {
    guide: guide.value?.name,
    space: props.space?.name
  });
});

function onClickBackButton() {
  if (props.from && JSON.parse(props.from).name) {
    router.push({
      name: JSON.parse(props.from).name,
      params: JSON.parse(props.from).params
    });
  } else {
    router.push({ name: 'guides', params: { key: props.space.id } });
  }
}
</script>

<template>
  <div class="page-main flex mx-auto justify-between">
    <LayoutSingle v-bind="$attrs">
      <template #content>
        <template v-if="loaded">
          <div class="px-4 md:px-0 mb-3 flex justify-between">
            <a class="text-color" @click="onClickBackButton()">
              <Icon name="back" size="22" class="!align-middle" />
              {{ backButtonText }}
            </a>
            <div class="ml-3">
              <UiDropdown
                top="2.5rem"
                right="1.5rem"
                class="float-right mr-2"
                @select="selectFromShareDropdown"
                @clickedNoDropdown="startShare(space, guide)"
                :items="sharingItems"
                :hideDropdown="sharingIsSupported"
              >
                <div class="pr-1 select-none">
                  <Icon name="upload" size="25" class="!align-text-bottom" />
                  Share
                </div>
              </UiDropdown>
              <UiDropdown
                top="2.5rem"
                right="1.3rem"
                class="float-right mr-2"
                @select="selectFromThreedotDropdown"
                v-if="isAdmin"
                :items="threeDotItems"
              >
                <div class="pr-3">
                  <UiLoading v-if="clientLoading" class="w-full" />
                  <Icon v-else name="threedots" size="25" />
                </div>
              </UiDropdown>
            </div>
          </div>
          <div class="px-4 md:px-0">
            <template v-if="loaded">
              <div class="flex justify-between">
                <div>
                  <h1 v-text="guide?.name" class="mb-2" />
                  <div class="mb-4">
                    <div class="flex justify-between">
                      <UiMarkdown :body="guide?.content" class="mb-6 w-[80%]" />
                    </div>
                  </div>
                </div>
                <div
                  class="guide-information mt-4"
                  v-if="isEthBlockchain || isOneBlockchain"
                >
                  <div class="space-y-1">
                    <div class="w-[180px] flex justify-between">
                      <div>
                        <b>{{ $t('author') }}</b>
                      </div>
                      <div>
                        <template v-for="author in guide.authors" :key="author">
                          <div>
                            <User
                              :address="author"
                              :profile="profiles[author]"
                              :space="space"
                              :guide="guide"
                              class="float-right"
                            />
                          </div>
                        </template>
                      </div>
                    </div>
                    <div>
                      <b>IPFS</b>
                      <a
                        :href="getIpfsUrl(guide.ipfs)"
                        target="_blank"
                        class="float-right"
                      >
                        #{{ guide.ipfs.slice(0, 7) }}
                        <Icon name="external-link" class="ml-1" />
                      </a>
                    </div>
                  </div>
                </div>
              </div>
              <Block
                :title="$t('guide.guideSteps')"
                :class="`mt-4`"
                slim
                v-if="guideLoaded && guide"
              >
                <div class="mt-4">
                  <GuideViewStepper
                    :activeStepId="activeStepId"
                    :goToNextStep="goToNextStep"
                    :goToPreviousStep="goToPreviousStep"
                    :guide="guide"
                    :guideResponse="guideResponseRef"
                    :guideSubmitting="guideSubmittingRef"
                    :selectAnswer="selectAnswer"
                    :setUserInput="setUserInput"
                    :submitGuide="submitGuide"
                  />
                </div>
              </Block>
            </template>
          </div>
        </template>
        <PageLoading v-else />
      </template>
    </LayoutSingle>
  </div>
  <teleport to="#modal">
    <ModalConfirm
      v-if="loaded"
      :open="modalOpen"
      @close="modalOpen = false"
      :space="space"
      :guide="guide"
      :id="spaceId"
    />
    <ModalGuideGuideExport
      v-if="loaded"
      :open="modalGuideExportOpen"
      @close="modalGuideExportOpen = false"
      :guide="guide"
    />
  </teleport>
</template>

<style scoped lang="scss">
.info-bar {
  box-shadow: var(--box-shadow);
  border-color: var(--border-color);
}
.guide-information {
  font-size: 0.9rem;
}
@media screen and (max-width: 991px) {
  .guide-information {
    display: none;
  }
}
</style>
