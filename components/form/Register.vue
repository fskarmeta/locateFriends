<script setup lang="ts">
import { Form } from "ant-design-vue";

const user = useSupabaseUser();
const client = useSupabaseClient();
const router = useRouter();
const useForm = Form.useForm;

const formState = reactive({
  username: "",
  latitude: null,
  longitude: null,
});

const { data: profile, refresh: refreshProfile } = await useAsyncData(
  "profiles",
  async () => {
    try {
      const { data: profile } = await client
        .from("profiles")
        .select("username, avatar_url, coordinates(*)")
        .eq("id", user.value.id)
        .single();
      return profile;
    } catch (e) {
      console.log("errrorr", e);
    }
  }
);

const onChangeCoords = ({ lat, lng }) => {
  formState.latitude = lat;
  formState.longitude = lng;
};

const rulesRef = reactive({
  username: [
    {
      required: true,
      message: "bra",
    },
  ],
  latitude: [
    {
      required: true,
      message: "bra",
      validator: () =>
        formState.latitude && formState.longitude
          ? Promise.resolve("nice")
          : Promise.reject("bra"),
    },
  ],
});

const { validate, validateInfos } = useForm(formState, rulesRef);

const onSubmit = () => {
  validate()
    .then(async () => {
      const body = toRaw(formState);
      const { error: profileError } = await client
        .from("profiles")
        .update({
          username: body.username,
        })
        .eq("id", user.value.id);
      const { error: coordinatesError } = await client
        .from("coordinates")
        .upsert({
          id: user.value.id,
          latitude: body.latitude,
          longitude: body.longitude,
        });
      if (!profileError && !coordinatesError) {
        return router.push({ name: "map" });
      }
    })
    .catch((error) => console.log(error));
};
</script>
<template>
  <div>
    <a-form :model="formState" name="registerForm" layout="vertical">
      <div class="flex place-items-center space-x-5">
        <a-form-item>
          <FormUpdateAvatar
            :avatarUrl="profile.avatar_url"
            :userId="user.id"
            @on-avatar-update="refreshProfile"
          />
        </a-form-item>
        <a-form-item label="Username" v-bind="validateInfos.username">
          <a-input v-model:value="formState.username" />
        </a-form-item>
      </div>
      <a-form-item
        label="Where do you live bro? Please select it on the map"
        v-bind="validateInfos.latitude"
      >
        <RegisterMap
          @onChangeCoords="onChangeCoords"
          class="w-screen md:w-500px h-100"
        />
      </a-form-item>

      <div class="w-full flex justify-center mt-5">
        <a-button type="primary" @click="onSubmit" html-type="submit"
          >Submit</a-button
        >
      </div>
    </a-form>
  </div>
</template>