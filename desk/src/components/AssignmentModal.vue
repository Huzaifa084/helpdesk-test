<template>
  <Dialog
    v-model="show"
    :options="{
      title: 'Assign To',
      size: 'xl',
    }"
  >
    <template #body-content>
      <SearchComplete
        class="form-control"
        doctype="HD Agent"
        :custom-filters="customFilters"
        :reset-input="true"
        @change="
          (option) => {
            addAssignee(option.value);
          }
        "
      >
        <template #item-prefix="{ option }">
          <UserAvatar class="mr-2" :name="option.value" size="sm" />
        </template>
        <template #item-label="{ option }">
          <Tooltip :text="option.value">
            {{ getUser(option.value).full_name }}
          </Tooltip>
        </template>
      </SearchComplete>
      <div class="mt-3 flex flex-wrap items-center gap-2">
        <Tooltip
          v-for="currentAssignee in assignees"
          :key="currentAssignee.name"
          :text="currentAssignee.name"
        >
          <Button
            :label="getUser(currentAssignee.name).full_name"
            theme="blue"
            variant="outline"
          >
            <template #prefix>
              <UserAvatar :name="currentAssignee.name" size="sm" />
            </template>
            <template #suffix>
              <FeatherIcon
                class="h-3.5"
                name="x"
                @click.stop="removeCurrentAssignee(currentAssignee.name)"
              />
            </template>
          </Button>
        </Tooltip>
      </div>
      <ErrorMessage v-if="error" class="mt-2" :message="error" />
    </template>
  </Dialog>
</template>

<script setup lang="ts">
import { SearchComplete, UserAvatar } from "@/components";
import { useAuthStore } from "@/stores/auth";
import { useUserStore } from "@/stores/user";
import { createResource } from "frappe-ui";
import { useOnboarding } from "frappe-ui/frappe";
import { computed, ref } from "vue";

const props = defineProps({
  assignees: {
    type: Array,
    required: true,
  },
  doctype: {
    type: String,
    required: true,
  },
  docname: {
    type: String,
    required: true,
  },
});

const show = defineModel();

const emit = defineEmits(["update"]);

const { getUser } = useUserStore();
const { updateOnboardingStep } = useOnboarding("helpdesk");
const { isManager } = useAuthStore();

const error = ref("");

const addAssignee = (value) => {
  error.value = "";
  createResource({
    url: "frappe.desk.form.assign_to.add",
    auto: true,
    params: {
      doctype: props.doctype,
      name: props.docname,
      assign_to: [value],
    },
    onSuccess: () => {
      emit("update");
      if (isManager) {
        updateOnboardingStep("assign_to_agent");
      }
    },
  });
  emit("update");
};

const removeCurrentAssignee = (value) => {
  createResource({
    url: "frappe.desk.form.assign_to.remove",
    auto: true,
    params: {
      doctype: props.doctype,
      name: props.docname,
      assign_to: value,
    },
    onSuccess: () => {
      emit("update");
    },
  });
};

const customFilters = computed(() => {
  const filters = {};
  filters["is_active"] = ["=", 1];
  if (Boolean(props.assignees?.length)) {
    filters["name"] = ["not in", [...props.assignees.map((a) => a.name)]];
  }
  return filters;
});
</script>
