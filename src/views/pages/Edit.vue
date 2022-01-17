<template>
  <gov-width-container>
    <ck-loader v-if="loading" />
    <template v-else>
      <vue-headful :title="`Help Yourself Sutton - Edit Page ${page.title}`" />

      <gov-back-link :to="{ name: 'pages-index' }">Back to pages</gov-back-link>
      <gov-main-wrapper>
        <page-form
          :page="page"
          :errors="form.$errors"
          :parent_id.sync="form.parent_id"
          :page_type.sync="form.page_type"
          :title.sync="form.title"
          :content.sync="form.content"
          :image_file_id.sync="form.image_file_id"
          :enabled.sync="form.enabled"
          @clear="form.$errors.clear($event)"
        />
      </gov-main-wrapper>

      <gov-section-break size="l" />

      <gov-grid-row>
        <gov-grid-column width="one-half">
          <gov-button v-if="form.$submitting" disabled type="submit"
            >Updating...</gov-button
          >
          <gov-button v-else @click="onSubmit" type="submit">Update</gov-button>

          <ck-submit-error v-if="form.$errors.any()" />
        </gov-grid-column>
        <gov-grid-column width="one-half">
          <ck-delete-button
            v-if="canDelete"
            resource="page"
            :endpoint="`/pages/${this.page.id}`"
            @deleted="onDelete"
          />
        </gov-grid-column>
      </gov-grid-row>
    </template>
  </gov-width-container>
</template>

<script>
import http from "@/http";
import Form from "@/classes/Form";
import PageForm from "./forms/PageForm";

export default {
  name: "EditPage",

  components: {
    PageForm
  },

  data() {
    return {
      loading: false,
      page: null,
      form: null
    };
  },
  computed: {
    updateButtonText() {
      return this.auth.isGlobalAdmin ? "Update" : "Request update";
    },
    canDelete() {
      return this.auth.isGlobalAdmin && this.page.children.length === 0;
    }
  },
  methods: {
    async fetchPage() {
      this.loading = true;

      const response = await http.get(`/pages/${this.$route.params.page}`);
      this.page = response.data.data;
      this.form = new Form({
        title: this.page.title,
        content: this.page.content,
        page_type: this.page.page_type,
        parent_id: this.page.parent ? this.page.parent.id : null,
        enabled: this.page.enabled,
        image_file_id: this.page.image ? this.page.image.id : null
      });

      this.loading = false;
    },
    async onSubmit() {
      await this.form.put(`/pages/${this.page.id}`, (config, data) => {
        // Remove any unchanged values.
        if (data.title === this.page.title) {
          delete data.title;
        }
        if (data.content === this.page.content) {
          delete data.content;
        }
        if (data.page_type === this.page.page_type) {
          delete data.page_type;
        }
        if (data.parent_id === this.page.parent_id) {
          delete data.parent_id;
        }
        if (data.enabled === this.page.enabled) {
          delete data.enabled;
        }

        // Remove the image from the request if null, or delete if false.
        if (data.image_file_id === null) {
          delete data.image_file_id;
        } else if (
          this.page.image &&
          data.image_file_id === this.page.image.id
        ) {
          delete data.image_file_id;
        } else if (data.image_file_id === false) {
          data.image_file_id = null;
        }
      });
      this.$router.push({
        name: "pages-index",
        query: { updated: true }
      });
    },
    onDelete() {
      this.$router.push({
        name: "pages-index"
      });
    }
  },
  created() {
    this.fetchPage();
  }
};
</script>

<style lang="scss" scoped></style>