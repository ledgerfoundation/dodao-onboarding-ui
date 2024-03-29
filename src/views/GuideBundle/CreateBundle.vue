<script setup lang="ts">
import Block from '@/components/Block.vue';
import GuideBundleGuideSelect from '@/components/GuideBundle/GuideSelect.vue';
import Icon from '@/components/Icon.vue';
import LayoutSingle from '@/components/Layout/Single.vue';
import ModalGuideGuideOrBundleType from '@/components/Modal/Guide/GuideOrBundleType.vue';
import ModalGuideCategory from '@/components/Modal/GuideCategory.vue';
import PageLoading from '@/components/PageLoading.vue';
import TextareaAutosize from '@/components/TextareaAutosize.vue';
import UiButton from '@/components/Ui/Button.vue';
import UiInput from '@/components/Ui/Input.vue';
import UiSidebarButton from '@/components/Ui/SidebarButton.vue';
import { useEditGuideBundle } from '@/composables/guide/useEditGuideBundle';
import { useApolloQuery } from '@/composables/useApolloQuery';
import { useClient } from '@/composables/useClient';
import { useInfiniteLoader } from '@/composables/useInfiniteLoader';
import { useModal } from '@/composables/useModal';
import { useStore } from '@/composables/useStore';
import { useWeb3 } from '@/composables/useWeb3';
import { GuidesQuery } from '@/graphql/guides.graphql';
import { setPageTitle } from '@/helpers/utils';
import {
  GuideBundlePublishStatus,
  GuideBundleType
} from '@dodao/onboarding-schemas/models/GuideBundleModel';
import { GuideModel } from '@dodao/onboarding-schemas/models/GuideModel';
import { SpaceModel } from '@dodao/onboarding-schemas/models/SpaceModel';
import { computed, inject, onMounted, PropType, ref, unref } from 'vue';
import { useRoute } from 'vue-router';

const props = defineProps({
  spaceId: String,
  space: { type: Object as PropType<SpaceModel>, required: true },
  from: String,
  uuid: String
});

const { web3, web3Account } = useWeb3();
const { clientLoading } = useClient();
const notify = inject('notify') as any;

const route = useRoute();

const bundleType = computed(() => route.params.bundleType);

const uuid = route.params.uuid;

const preview = ref(false);

const modalCategoryOpen = ref(false);

const modalGuideOrBundleTypeOpen = ref<boolean>(false);

const {
  addEmptyBundleGuideInput,
  guideBundleCreating,
  guideBundleLoaded,
  guideBundleRef: guideBundle,
  guideBundleErrors,
  handleSubmit,
  initialize,
  moveGuideUp,
  moveGuideDown,
  removeGuideInput,
  selectGuide
} = useEditGuideBundle(uuid as string, props.space, notify);
const { modalAccountOpen } = useModal();
const { store } = useStore();
const { apolloQuery } = useApolloQuery();
const { loadBy, stopLoadingMore } = useInfiniteLoader();

const form = ref(guideBundle);

const bundleGuides = computed(() => {
  console.log('form.value.bundleGuides', form.value.bundleGuides);
  return form.value.bundleGuides;
});

const isValid = computed(() => {
  return !clientLoading.value && !web3.value.authLoading;
});

function clickSubmit() {
  !web3Account.value ? (modalAccountOpen.value = true) : handleSubmit();
}

const errors = unref(guideBundleErrors);

const uploadThumbnailLoading = ref(false);
const uploadSocialShareImageLoading = ref(false);

const loadingGuides = ref(false);
const guides = ref<GuideModel[]>([]);

const categoriesString = computed(() => {
  return form.value.categories ? form.value.categories.join(', ') : '';
});

const guidesMap = computed(() => {
  return Object.fromEntries(
    form.value.bundleGuides.map(guide => [guide.uuid, guide])
  );
});

const bundleHasErrors = computed(() => {
  return Object.values(guideBundleErrors.value).filter(v => !!v).length > 0;
});

function handleSubmitAddCategories(categories) {
  guideBundle.value.categories = categories;
}

function setThumbnailUrl(url) {
  if (typeof url === 'string') form.value.thumbnail = url;
}

function setSocialShareImageUrl(url) {
  if (typeof url === 'string') form.value.socialShareImage = url;
}

function setUploadLoading(s) {
  uploadThumbnailLoading.value = s;
}

function setUploadSocialShareImage(s) {
  uploadSocialShareImageLoading.value = s;
}

function inputError(field: string) {
  return errors[field];
}

async function loadGuides(skip = 0) {
  loadingGuides.value = true;
  const guidesResult = await apolloQuery(
    {
      query: GuidesQuery,
      variables: {
        first: loadBy,
        skip,
        space: props.spaceId
      }
    },
    'guides'
  );
  stopLoadingMore.value = guidesResult?.length < loadBy;

  store.space.guides = guidesResult;
  guides.value = guidesResult;
  loadingGuides.value = false;
}

onMounted(async () => {
  setPageTitle('page.title.guide.create', { space: props.space?.name });
  loadGuides(store.space.guides.length);
  await initialize();
});

function selectGuideOrBundleType(type: GuideBundleType) {
  guideBundle.value.bundleType = type;
}

function selectPublishStatus(status) {
  form.value.publishStatus = status;
}

const guideBundleStatuses = [
  {
    text: 'Live',
    action: GuideBundlePublishStatus.Live
  },
  {
    text: 'Draft',
    action: GuideBundlePublishStatus.Draft
  }
];
</script>

<template>
  <LayoutSingle v-bind="$attrs">
    <template #content>
      <div class="px-4 md:px-0 overflow-hidden">
        <router-link
          :to="
            guideBundle.id
              ? {
                  name: 'guideBundle',
                  params: { bundleType, key: space.id, uuid: guideBundle.uuid }
                }
              : { name: 'guideBundles', params: { bundleType } }
          "
          class="text-color"
        >
          <Icon name="back" size="22" class="!align-middle" />
          {{ guideBundle.id ? guideBundle.name : 'Back to Guide Bundles' }}
        </router-link>

        <UiSidebarButton
          v-if="preview"
          @click="preview = false"
          class="float-right"
        >
          <Icon name="back" size="18" />
        </UiSidebarButton>
      </div>
      <template v-if="guideBundleLoaded">
        <Block :title="$t('guideBundle.create.basicInfo')" :class="`mt-4`">
          <div class="mb-2">
            <UiInput
              v-model="form.name"
              :error="inputError('name')"
              maxlength="32"
            >
              <template v-slot:label
                >{{ $t(`guideBundle.create.name`) }}*</template
              >
            </UiInput>
            <UiInput
              v-model="form.thumbnail"
              placeholder="e.g. https://example.com/guide-bundle.png"
              :error="inputError('avatar')"
            >
              <template v-slot:label>
                {{ $t('guideBundle.thumbnail') }}
              </template>
              <template v-slot:info>
                <Upload
                  class="!ml-2"
                  @input="setThumbnailUrl"
                  @loading="setUploadLoading"
                >
                  {{ $t('upload') }}
                </Upload>
              </template>
            </UiInput>
            <UiInput
              v-model="form.socialShareImage"
              placeholder="e.g. https://example.com/-bundle.png ideally 800px by 418px"
              :error="inputError('socialShareImage')"
            >
              <template v-slot:label>
                {{ $t('guideBundle.socialShareImage') }}
              </template>
              <template v-slot:info>
                <Upload
                  class="!ml-2"
                  @input="setSocialShareImageUrl"
                  @loading="setUploadSocialShareImage"
                >
                  {{ $t('upload') }}
                </Upload>
              </template>
            </UiInput>
            <UiInput
              v-model="form.discordWebhook"
              placeholder="e.g. https://discord.com/api/webhooks/xxxxxxxxxx"
              :error="inputError('discordWebhook')"
            >
              <template v-slot:label>
                {{ $t('guideBundle.discordWebhook') }}
              </template>
            </UiInput>
            <UiInput
              @click="modalCategoryOpen = true"
              :class="{ 'mt-6': !categoriesString }"
            >
              <template v-slot:label>
                {{ $t(`settings.categories`) }}
              </template>
              <template v-slot:selected>
                <span class="capitalize">
                  {{ categoriesString }}
                </span>
              </template>
            </UiInput>
            <UiInput @click="modalGuideOrBundleTypeOpen = true">
              <template v-slot:label>
                {{ $t(`guideBundle.bundleType`) }}
              </template>
              <template v-slot:selected>
                <span class="capitalize">
                  {{ $tc('navigation.' + form.bundleType) }}
                </span>
              </template>
            </UiInput>
            <UiInput
              v-model="form.excerpt"
              :error="guideBundleErrors.excerpt"
              :placeholder="$t(`guideBundle.create.excerpt`)"
              maxlength="64"
            >
              <template v-slot:label
                >{{ $t(`guideBundle.create.excerpt`) }}*</template
              >
            </UiInput>
            <div class="status-wrapper pt-3">
              <UiDropdown
                top="2.5rem"
                right="2.5rem"
                class="mr-2 w-[5rem] status-drop-down"
                @select="selectPublishStatus($event)"
                :items="guideBundleStatuses"
              >
                <div class="pr-1 select-none">
                  {{ form.publishStatus === 'Live' ? 'Live' : 'Draft' }}
                </div>
              </UiDropdown>
              <div
                class="input-label text-color mr-2 whitespace-nowrap absolute forceFloat"
              >
                Publish Status*
              </div>
            </div>
            <UiButton
              class="w-full h-96 mb-4 mt-6 px-[16px] flex items-center"
              style="height: max-content"
              :class="{ '!border-red': guideBundleErrors.content }"
            >
              <TextareaAutosize
                :value="guideBundle.content"
                :placeholder="$t(`guideBundle.create.content`)"
                :minHeight="200"
                class="input w-full text-left"
                style="font-size: 18px"
                @update:modelValue="form.content = $event"
              />
              <i
                class="iconfont iconwarning !text-red"
                data-v-abc9f7ae=""
                v-if="guideBundleErrors.content"
              ></i>
            </UiButton>
          </div>
        </Block>
        <Block :title="$t('guideBundle.create.guidesInBundle')" :class="`mt-4`">
          <template v-for="bundleGuide in bundleGuides" :key="bundleGuide.uuid">
            <GuideBundleGuideSelect
              :bundle-input="guideBundle"
              :guide-errors="
                guideBundleErrors?.bundleGuides?.[bundleGuide.uuid]
              "
              :guide-input="bundleGuide"
              :guides="guides"
              :move-guide-down="moveGuideDown"
              :move-guide-up="moveGuideUp"
              :remove-guide-input="removeGuideInput"
              :select-guide="selectGuide"
            />
          </template>
        </Block>
        <div
          v-if="bundleHasErrors"
          class="!text-red flex text-center justify-center mb-2 align-baseline"
        >
          <i class="iconfont iconwarning !text-red" data-v-abc9f7ae=""></i>
          <span class="ml-1">Fix errors to proceed</span>
        </div>
        <UiButton
          @click="addEmptyBundleGuideInput"
          :disabled="
            clientLoading ||
            !guideBundleLoaded ||
            guideBundleCreating ||
            bundleGuides.length >= guides.length
          "
          class="block w-full"
          primary
        >
          {{ $t('guideBundle.create.addGuide') }}
        </UiButton>
        <UiButton
          @click="clickSubmit"
          :disabled="!isValid"
          :loading="clientLoading || !guideBundleLoaded || guideBundleCreating"
          class="block w-full mt-4"
          variant="contained"
          primary
        >
          {{ $t('create.publish') }}
        </UiButton>
      </template>
      <PageLoading v-else />
    </template>
  </LayoutSingle>
  <teleport to="#modal">
    <ModalGuideCategory
      :open="modalCategoryOpen"
      :categories="guideBundle.categories"
      @close="modalCategoryOpen = false"
      @add="handleSubmitAddCategories"
    />
    <ModalGuideGuideOrBundleType
      :open="modalGuideOrBundleTypeOpen"
      :selected-type="form.bundleType"
      :for-bundle-type="true"
      @close="modalGuideOrBundleTypeOpen = false"
      @selectGuideOrBundleType="selectGuideOrBundleType"
    />
  </teleport>
</template>
<style scoped lang="scss">
.status-wrapper {
  border-bottom: 1px solid var(--border-color);
}
.forceFloat {
  transform: translatey(-44px);
  @apply text-xs;
  transition: transform 0.1s linear, font-size 0.1s linear;
}

.status-drop-down {
  color: var(--link-color);
}
</style>
