<template>
  <gov-width-container>
    <vue-headful :title="`${appName} - Login`" />

    <gov-back-link :to="{ name: 'dashboard' }">Back to dashboard</gov-back-link>
    <gov-main-wrapper>
      <gov-grid-row>
        <gov-grid-column width="two-thirds">
          <gov-heading size="xl">Login</gov-heading>

          <template v-if="!validateRequest">
            <gov-body size="l">
              Click below to login to the {{ appName }} admin portal:
            </gov-body>

            <gov-button :href="loginUrl">Login</gov-button>

            <gov-body size="s">
              For security reasons, you will be automatically logged out after
              {{ sessionMinutes }} minutes.
            </gov-body>
          </template>
        </gov-grid-column>
      </gov-grid-row>
    </gov-main-wrapper>
  </gov-width-container>
</template>

<script>
import Auth from "@/classes/Auth";

export default {
  name: "Login",
  data() {
    return {
      accessToken:
        Auth.parseQueryString(window.location.href).access_token || null,
      expiresIn: Auth.parseQueryString(window.location.href).expires_in || null
    };
  },
  computed: {
    loginUrl() {
      return Auth.authorizeUrl;
    },
    validateRequest() {
      if (this.accessToken === null) {
        return false;
      }

      if (this.expiresIn === null) {
        return false;
      }

      return true;
    }
  },
  methods: {
    async login() {
      await Auth.login(this.accessToken, this.expiresIn);
      this.$root.$emit("login");
      this.$router.push({ name: "dashboard" });
    }
  },
  created() {
    if (this.validateRequest) {
      this.login();
    }
  }
};
</script>
