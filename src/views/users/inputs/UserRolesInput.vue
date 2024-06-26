<template>
  <div>
    <gov-form-group
      v-for="(role, index) in roles"
      :key="role.index"
      :invalid="errors.has('roles')"
    >
      <gov-inset-text>
        <gov-error-message
          v-if="errors.has(`roles.${index}`)"
          v-text="errors.get(`roles.${index}`)"
          :for="`roles.${index}`"
        />

        <!-- Role -->
        <ck-select-input
          :value="roles[index].role"
          @input="onRoleInput({ key: index, value: $event })"
          :id="`roles_${index}_role`"
          label="User type"
          :options="userRoleOptions"
          :error="errors.get(`roles.${index}.role`)"
        />
        <!-- /Role -->

        <template v-if="showOrganisationSelect(index)">
          <!-- Organisation -->
          <gov-form-group v-if="loadingOrganisations">
            <ck-loader />
          </gov-form-group>
          <ck-select-input
            v-else
            :value="roles[index].organisation_id"
            @input="onOrganisationInput({ key: index, value: $event })"
            :id="`roles_${index}_organisation_id`"
            label="Organisation"
            :options="organisations"
            :error="errors.get(`roles.${index}.organisation_id`)"
          />
          <!-- /Organisation -->

          <template v-if="showServiceSelect(index)">
            <!-- Service -->
            <gov-form-group
              v-if="!services.hasOwnProperty(roles[index].organisation_id)"
            >
              Select an organisation
            </gov-form-group>
            <gov-form-group
              v-else-if="services[roles[index].organisation_id].loading"
            >
              <ck-loader />
            </gov-form-group>
            <ck-select-input
              v-else
              :value="roles[index].service_id"
              @input="onServiceInput({ key: index, value: $event })"
              :id="`roles_${index}_service_id`"
              label="Service"
              :options="services[roles[index].organisation_id].items"
              :error="errors.get(`roles.${index}.service_id`)"
            />
            <!-- /Service -->
          </template>
        </template>

        <gov-section-break size="m" />

        <gov-button @click="onRemovePermission(index)" error>Remove</gov-button>
      </gov-inset-text>
    </gov-form-group>

    <gov-inset-text v-if="roles.length === 0">
      <gov-body>No permissions</gov-body>
    </gov-inset-text>

    <gov-button @click="onAddPermission">Add permission</gov-button>

    <gov-error-message
      v-if="errors.has('roles')"
      v-text="errors.get('roles')"
      for="roles"
    />
  </div>
</template>

<script>
import Auth from "@/classes/Auth";

export default {
  name: "UserRolesInput",
  model: {
    prop: "roles",
    event: "input"
  },
  props: {
    roles: {
      required: true,
      type: Array
    },
    errors: {
      required: true,
      type: Object
    }
  },
  data() {
    return {
      auth: Auth,
      roleIndex: 0,
      loadingOrganisations: false,
      organisations: [],
      services: {}
    };
  },
  computed: {
    roleOptions() {
      const roles = this.auth.roles.slice();
      roles.unshift({ text: "Please select", value: null, disabled: true });
      return roles;
    },
    userRoleOptions() {
      const highestRole = this.auth.displayHighestRole(this.auth.user.roles);
      return this.roleOptions.filter(option => {
        if (highestRole === "Super Admin") {
          return true;
        } else if (highestRole === "Global Admin") {
          return option.value !== "Super Admin";
        } else if (highestRole === "Organisation Admin") {
          return !["Super Admin", "Global Admin", "Content Admin"].includes(
            option.value
          );
        } else if (highestRole === "Service Admin") {
          return ![
            "Super Admin",
            "Global Admin",
            "Content Admin",
            "Organisation Admin"
          ].includes(option.value);
        } else {
          return false;
        }
      });
    }
  },
  watch: {
    roles: {
      handler: function(newRoles, oldRoles) {
        if (JSON.stringify(newRoles) === JSON.stringify(oldRoles)) {
          return;
        }

        newRoles = newRoles.map(role => ({ ...role }));

        newRoles.forEach(role => {
          // If the role uses a service, then lazy load the services and cache them.
          if (role.role === "Service Admin" || role.role === "Service Worker") {
            if (role.organisation_id !== null) {
              this.cacheServices(role.organisation_id);
            }
          }

          // Reset the organisation adn service ID when no longer applies to role.
          switch (role.role) {
            case "Super Admin":
            case "Global Admin":
            case "Content Admin": {
              role.organisation_id = null;
              role.service_id = null;
              this.$emit("input", newRoles);
              break;
            }
            case "Organisation Admin": {
              role.service_id = null;
              this.$emit("input", newRoles);
              break;
            }
          }
        });
      },
      deep: true
    }
  },
  methods: {
    cloneRoles() {
      return this.roles.map(role => ({ ...role }));
    },
    showOrganisationSelect(index) {
      return ["Organisation Admin", "Service Admin", "Service Worker"].includes(
        this.roles[index].role
      );
    },
    showServiceSelect(index) {
      return ["Service Admin", "Service Worker"].includes(
        this.roles[index].role
      );
    },
    onRoleInput({ key, value }) {
      this.$emit("clear", `roles.${key}.role`);

      let roles = this.cloneRoles();
      roles[key].role = value;
      this.$emit("input", roles);
    },
    onOrganisationInput({ key, value }) {
      this.$emit("clear", `roles.${key}.organisation_id`);

      let roles = this.cloneRoles();
      roles[key].organisation_id = value;
      this.$emit("input", roles);
    },
    onServiceInput({ key, value }) {
      this.$emit("clear", `roles.${key}.service_id`);

      let roles = this.cloneRoles();
      roles[key].service_id = value;
      this.$emit("input", roles);
    },
    onAddPermission() {
      // The role index is purely used for Vue to keep track when looping.
      let roles = this.cloneRoles();
      roles.push({
        role: null,
        organisation_id: null,
        service_id: null,
        index: this.roleIndex
      });
      this.$emit("input", roles);
      this.$emit("clear", "roles");
      this.roleIndex++;
    },
    onRemovePermission(index) {
      this.$emit("clear", `roles.${index}.role`);
      this.$emit("clear", `roles.${index}.organisation_id`);
      this.$emit("clear", `roles.${index}.service_id`);
      let roles = this.cloneRoles();
      this.$delete(roles, index);
      this.$emit("input", roles);
    },
    async cacheOrganisations() {
      this.loadingOrganisations = true;
      this.organisations = [
        { value: null, text: "Please select", disabled: true }
      ];

      let organisations = await this.fetchAll("/organisations");
      organisations = organisations.map(organisation => {
        return {
          value: organisation.id,
          text: organisation.name
        };
      });
      if (!this.auth.isSuperAdmin && !this.auth.isGlobalAdmin) {
        organisations = organisations.filter(organisation => {
          return this.auth.user.roles.find(
            role =>
              role.organisation_id === organisation.value &&
              role.role !== "Service Worker"
          );
        });
      }

      this.organisations = [...this.organisations, ...organisations];

      this.loadingOrganisations = false;
    },
    async cacheServices(organisationIds) {
      organisationIds = Array.isArray(organisationIds)
        ? organisationIds
        : [organisationIds];

      for (let organisationId of organisationIds) {
        // Skip if the services have already been cached.
        if (this.services.hasOwnProperty(organisationId)) {
          continue;
        }

        // Set the initial object.
        this.services[organisationId] = {
          items: [{ value: null, text: "Please select", disabled: true }],
          loading: true
        };
      }

      // Fetch all services for the organisations
      let services = await this.fetchAll(
        "/services/index",
        { "filter[organisation_id]": organisationIds.join(",") },
        "POST"
      );

      // Combine the local cache with the remote objects
      for (let organisationId of organisationIds) {
        let orgServices = [
          ...this.services[organisationId].items,
          ...services
            .filter(
              service =>
                service.organisation_id === organisationId &&
                !this.services[organisationId].items.find(
                  orgService => orgService.value === service.id
                )
            )
            .map(service => ({ value: service.id, text: service.name }))
        ];

        // Remove any services the auth user is not an admin for
        if (
          !this.auth.isSuperAdmin &&
          !this.auth.isGlobalAdmin &&
          !this.auth.isOrganisationAdmin(organisationId)
        ) {
          orgServices = orgServices.filter(service =>
            this.auth.isServiceAdmin(service.value)
          );
        }

        this.services[organisationId].items = orgServices;

        this.services[organisationId].loading = false;
      }

      this.$forceUpdate();
    }
  },
  created() {
    this.cacheOrganisations();

    let organisationIds = {};

    this.roles.forEach(role => {
      if (role.hasOwnProperty("organisation_id")) {
        organisationIds[role.organisation_id] = null;
      }
    });

    this.cacheServices(Object.keys(organisationIds));
  }
};
</script>
