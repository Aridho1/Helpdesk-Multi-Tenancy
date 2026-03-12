<script setup lang="ts">
import { ref, onMounted } from "vue";
import { useRouter } from "vue-router";
import DataTable from "primevue/datatable";
import Column from "primevue/column";
import Button from "primevue/button";
import Tag from "primevue/tag";
import Dialog from "primevue/dialog";
import { tenantApi, type Tenant } from "@/api/tenant";
import { useToast } from "@/composables/useToast";
import { logger } from "@/utils/logger";

const router = useRouter();
const toast = useToast();

const tenants = ref<Tenant[]>([]);
const isLoading = ref(false);

const isImageModalVisible = ref(false);
const selectedImage = ref("");

const loadTenants = async () => {
    isLoading.value = true;
    try {
        const data = await tenantApi.list();
        tenants.value = data;
    } catch (error) {
        logger.error("Failed to load tenants", error);
        toast.error("Failed to load tenants");
    } finally {
        isLoading.value = false;
    }
};

onMounted(() => {
    loadTenants();
});

const getLogoUrl = (url: string) => {
    if (!url) return "";
    if (url.startsWith("http") || url.startsWith("data:")) return url;
    if (url.startsWith("/uploads/")) {
        const apiBase = import.meta.env.VITE_API_BASE_URL || "http://localhost:8080/api/v1";
        const origin = new URL(apiBase).origin;
        return `${origin}${url}`;
    }
    return url;
};

const openImageModal = (url: string) => {
    selectedImage.value = getLogoUrl(url);
    isImageModalVisible.value = true;
};

const handleEdit = (tenant: Tenant) => {
    router.push(`/admin/tenants/${tenant.id}/edit`);
};

const handleToggleStatus = async (tenant: Tenant) => {
    try {
        const newStatus = !tenant.is_active;
        await tenantApi.updateStatus(tenant.id, newStatus);
        tenant.is_active = newStatus;
        toast.success(`Tenant ${newStatus ? "activated" : "deactivated"} successfully`);
    } catch (error) {
        logger.error("Failed to update tenant status", error);
        toast.error("Failed to update tenant status");
    }
};

const handleCreate = () => {
    router.push("/admin/tenants/create");
};
</script>

<template>
    <div class="page-container">
        <div class="page-header">
            <div>
                <h1 class="page-title">Tenant Management</h1>
                <p class="page-subtitle">Manage system tenants and companies</p>
            </div>
            <Button label="Add Tenant" icon="pi pi-plus" @click="handleCreate" />
        </div>

        <div class="card">
            <DataTable :value="tenants" :loading="isLoading" stripedRows responsiveLayout="scroll">
                <template #empty>No tenants found.</template>

                <Column field="name" header="Company Name" sortable>
                    <template #body="{ data }">
                        <div class="c-name">
                            <img v-if="data.logo_url" :src="getLogoUrl(data.logo_url)" alt="Logo" class="logo-tenant clickable" @click="openImageModal(data.logo_url)" />
                            <div v-else class="logo-fallback">
                                {{ data.name.substring(0, 2).toUpperCase() }}
                            </div>
                            <span class="tenant-name">{{ data.name }}</span>
                        </div>
                    </template>
                </Column>

                <Column field="slug" header="Slug" sortable>
                    <template #body="{ data }">
                        <code class="slug-code">{{ data.slug }}</code>
                    </template>
                </Column>

                <Column field="primary_color" header="Brand Color">
                    <template #body="{ data }">
                        <div class="color-display">
                            <div class="color-box" :style="{ backgroundColor: data.primary_color }"></div>
                            <span class="color-text">{{ data.primary_color }}</span>
                        </div>
                    </template>
                </Column>

                <Column field="is_active" header="Status" sortable>
                    <template #body="{ data }">
                        <Tag :severity="data.is_active ? 'success' : 'danger'" :value="data.is_active ? 'Active' : 'Inactive'" />
                    </template>
                </Column>

                <Column header="Actions" alignFrozen="right" :exportable="false" style="min-width: 8rem">
                    <template #body="{ data }">
                        <div class="action-buttons">
                            <Button icon="pi pi-pencil" severity="secondary" text rounded aria-label="Edit" @click="handleEdit(data)" />
                            <Button
                                :icon="data.is_active ? 'pi pi-ban' : 'pi pi-check-circle'"
                                :severity="data.is_active ? 'danger' : 'success'"
                                text
                                rounded
                                :aria-label="data.is_active ? 'Deactivate' : 'Activate'"
                                @click="handleToggleStatus(data)"
                                v-tooltip.top="data.is_active ? 'Deactivate' : 'Activate'" />
                        </div>
                    </template>
                </Column>
            </DataTable>
        </div>

        <Dialog v-model:visible="isImageModalVisible" modal header="Image Preview" :style="{ width: '50vw' }" :breakpoints="{ '1199px': '75vw', '575px': '90vw' }" dismissableMask>
            <div class="modal-image-container">
                <img :src="selectedImage" alt="Preview" class="modal-image" />
            </div>
        </Dialog>
    </div>
</template>

<style scoped>
.page-container {
    padding: 2rem;
    max-width: 1200px;
    margin: 0 auto;
}

.page-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 2rem;
}

.page-title {
    font-size: 1.875rem;
    font-weight: 600;
    color: var(--text-primary);
    margin: 0 0 0.5rem 0;
}

.page-subtitle {
    color: var(--text-secondary);
    margin: 0;
}

.card {
    background: white;
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
    padding: 1.5rem;
    border: 1px solid var(--border-color);
}

.c-name {
    display: flex;
    align-items: center;
    gap: 0.5rem;
}

.logo-tenant {
    width: 2rem;
    height: 2rem;
    object-fit: contain;
    border-radius: 0.25rem;
    background-color: #f3f4f6; /* gray-100 */
    padding: 0.25rem;
}

.logo-tenant.clickable {
    cursor: pointer;
    transition: transform 0.2s ease;
}

.logo-tenant.clickable:hover {
    transform: scale(1.05);
}

.logo-fallback {
    width: 2rem;
    height: 2rem;
    border-radius: 0.25rem;
    background-color: #e5e7eb; /* gray-200 */
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 0.75rem;
    font-weight: 700;
    color: #6b7280; /* gray-500 */
}

.tenant-name {
    font-weight: 500;
}

.slug-code {
    background-color: #f3f4f6; /* gray-100 */
    padding: 0.25rem 0.5rem;
    border-radius: 0.25rem;
    font-size: 0.875rem;
    color: #374151; /* gray-700 */
}

.color-display {
    display: flex;
    align-items: center;
    gap: 0.5rem;
}

.color-box {
    width: 1.5rem;
    height: 1.5rem;
    border-radius: 0.25rem;
    border: 1px solid #e5e7eb; /* border-gray-200 */
}

.color-text {
    font-size: 0.875rem;
    color: #4b5563; /* gray-600 */
}

.action-buttons {
    display: flex;
    gap: 0.5rem;
}

.modal-image-container {
    display: flex;
    justify-content: center;
    align-items: center;
    padding: 1rem;
    background-color: #f9fafb;
    border-radius: 8px;
}

.modal-image {
    max-width: 100%;
    max-height: 70vh;
    object-fit: contain;
    border-radius: 4px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}
</style>
