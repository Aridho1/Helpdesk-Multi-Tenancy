<script setup lang="ts">
import { ref, computed, onMounted } from "vue";
import { useI18n } from "vue-i18n";
import { attachmentsApi } from "@/api/attachments";
import type { Attachment } from "@/api/types";
import { logger } from "@/utils/logger";
import Dialog from "primevue/dialog";

const { t } = useI18n();

interface Props {
    attachmentId: number;
    attachment?: { name?: string; extension?: string; url?: string };
}

const props = defineProps<Props>();

const attachmentData = ref<Attachment | null>(null);
const isLoading = ref(false);

const isMediaModalVisible = ref(false);
const selectedMediaUrl = ref("");
const selectedMediaType = ref<"image" | "video">("image");

const openMediaModal = (url: string, type: "image" | "video", event?: Event) => {
    if (event) {
        event.preventDefault();
        event.stopPropagation();
    }
    selectedMediaUrl.value = url;
    selectedMediaType.value = type;
    isMediaModalVisible.value = true;
};

onMounted(async () => {
    if (!props.attachment?.url) {
        isLoading.value = true;
        try {
            const data = await attachmentsApi.getDetail(props.attachmentId);
            logger.debug("Fetched attachment data", data);
            if (data && data.url) {
                attachmentData.value = data;
            } else {
                logger.warn("Fetched attachment data but no URL found", data);
                attachmentData.value = data;
            }
        } catch (err) {
            logger.error("Failed to fetch attachment detail", err);
            if (props.attachment) {
                attachmentData.value = props.attachment as Attachment;
            }
        } finally {
            isLoading.value = false;
        }
    } else {
        attachmentData.value = props.attachment as Attachment;
        logger.debug("Using provided attachment data", attachmentData.value);
    }
});

const attachment = computed(() => {
    return attachmentData.value || props.attachment;
});

const displayName = computed(() => {
    const rawName = attachment.value?.name || `Attachment #${props.attachmentId}`;
    const ext = attachment.value?.extension?.toLowerCase();

    if (!ext) return rawName;

    const lowerName = rawName.toLowerCase();
    const suffix = `.${ext}`;

    if (lowerName.endsWith(suffix)) {
        return rawName.slice(0, -suffix.length);
    }

    return rawName;
});

const attachmentUrl = computed(() => {
    if (attachment.value?.url) {
        const invgateBaseUrl = "https://support.armmada.id";
        const url = attachment.value.url.startsWith("/") ? attachment.value.url : `/${attachment.value.url}`;
        return `${invgateBaseUrl}${url}`;
    }
    return null;
});

const isImage = computed(() => {
    const ext = attachment.value?.extension?.toLowerCase() || "";
    return ["jpg", "jpeg", "png", "gif", "webp", "svg", "bmp"].includes(ext);
});

const isPdf = computed(() => {
    const ext = attachment.value?.extension?.toLowerCase() || "";
    return ext === "pdf";
});

const isVideo = computed(() => {
    const ext = attachment.value?.extension?.toLowerCase() || "";
    return ["mp4", "webm", "ogg", "mov", "avi"].includes(ext);
});

const buildFullUrl = (url: string): string => {
    const invgateBaseUrl = "https://support.armmada.id";
    const normalizedUrl = url.startsWith("/") ? url : `/${url}`;
    return `${invgateBaseUrl}${normalizedUrl}`;
};

const handleViewClick = async (event: Event) => {
    event.preventDefault();
    event.stopPropagation();

    logger.debug("View button clicked", {
        attachmentUrl: attachmentUrl.value,
        attachment: attachment.value,
        attachmentData: attachmentData.value,
        propsAttachment: props.attachment,
    });

    let urlToOpen: string | null = null;

    if (attachmentUrl.value) {
        urlToOpen = attachmentUrl.value;
        logger.debug("Using URL from computed", urlToOpen);
    } else {
        const currentAttachment = attachment.value;
        if (currentAttachment?.url) {
            urlToOpen = buildFullUrl(currentAttachment.url);
            logger.debug("Using constructed URL from attachment object", urlToOpen);
        }
    }

    if (urlToOpen) {
        const newWindow = window.open(urlToOpen, "_blank", "noopener,noreferrer");
        if (!newWindow) {
            logger.warn("Popup blocked, trying alternative method");
            const link = document.createElement("a");
            link.href = urlToOpen;
            link.target = "_blank";
            link.rel = "noopener noreferrer";
            document.body.appendChild(link);
            link.click();
            document.body.removeChild(link);
        }
        return;
    }

    logger.debug("No URL found, attempting to fetch attachment detail");
    const newWindow = window.open("about:blank", "_blank", "noopener,noreferrer");

    if (!newWindow) {
        alert("Please allow popups for this site to view attachments.");
        return;
    }

    isLoading.value = true;
    try {
        const data = await attachmentsApi.getDetail(props.attachmentId);
        logger.debug("Fetched attachment data on click", data);

        if (data?.url) {
            attachmentData.value = data;
            const fullUrl = buildFullUrl(data.url);
            logger.debug("Opening URL from fresh fetch", fullUrl);
            newWindow.location.href = fullUrl;
        } else {
            logger.error("Fetched attachment but no URL in response", data);
            newWindow.close();
            alert("Attachment URL is not available. The attachment may not have a valid URL.");
        }
    } catch (err) {
        logger.error("Failed to fetch attachment detail on click", err);
        newWindow.close();
        alert("Failed to load attachment. Please try again later.");
    } finally {
        isLoading.value = false;
    }
};
</script>

<template>
    <bx-tile :id="`attachmentPreviewTile-${attachmentId}`" class="attachment-preview">
        <div v-if="isImage" class="attachment-file">
            <div class="attachment-info">
                <div v-if="attachmentUrl && !isLoading" class="media-thumbnail-container" @click.stop="openMediaModal(attachmentUrl, 'image')">
                    <img :src="attachmentUrl" class="media-thumbnail" alt="Thumbnail" />
                </div>
                <span v-else class="attachment-icon" aria-hidden="true">
                    <i class="pi pi-image"></i>
                </span>
                <div class="attachment-details">
                    <span class="attachment-name">
                        {{ displayName }}
                    </span>
                    <span v-if="attachment?.extension" class="attachment-extension">.{{ attachment.extension }}</span>
                </div>
            </div>
            <button
                v-if="attachmentUrl && !isLoading"
                :id="`attachmentPreviewViewLink-${attachmentId}`"
                type="button"
                class="view-button"
                @click.prevent.stop="openMediaModal(attachmentUrl, 'image', $event)">
                View Image
            </button>
            <button
                v-else
                :id="`attachmentPreviewViewBtn-${attachmentId}`"
                type="button"
                class="view-button"
                :disabled="isLoading"
                @click.prevent.stop="handleViewClick"
                @mousedown.prevent.stop
                @mouseup.prevent.stop
                @touchstart.prevent.stop
                @touchend.prevent.stop>
                {{ isLoading ? t("common.loading") : t("tickets.detailPage.attachment.preview") }}
            </button>
        </div>

        <div v-else-if="isPdf" class="attachment-file">
            <div class="attachment-info">
                <span class="attachment-icon" aria-hidden="true">
                    <i class="pi pi-file"></i>
                </span>
                <div class="attachment-details">
                    <span class="attachment-name">
                        {{ displayName }}
                    </span>
                    <span v-if="attachment?.extension" class="attachment-extension">.{{ attachment.extension }}</span>
                </div>
            </div>
            <a v-if="attachmentUrl && !isLoading" :id="`attachmentPreviewViewLink-${attachmentId}`" :href="attachmentUrl" target="_blank" rel="noopener noreferrer" class="view-button" @click.stop>
                View PDF
            </a>
            <button
                v-else
                :id="`attachmentPreviewViewBtn-${attachmentId}`"
                type="button"
                class="view-button"
                :disabled="isLoading"
                @click.prevent.stop="handleViewClick"
                @mousedown.prevent.stop
                @mouseup.prevent.stop
                @touchstart.prevent.stop
                @touchend.prevent.stop>
                {{ isLoading ? t("common.loading") : t("tickets.detailPage.attachment.preview") }}
            </button>
        </div>

        <div v-else-if="isVideo" class="attachment-file">
            <div class="attachment-info">
                <div v-if="attachmentUrl && !isLoading" class="media-thumbnail-container" @click.stop="openMediaModal(attachmentUrl, 'video')">
                    <video :src="attachmentUrl" class="media-thumbnail"></video>
                    <div class="play-overlay"><i class="pi pi-play"></i></div>
                </div>
                <span v-else class="attachment-icon" aria-hidden="true">
                    <i class="pi pi-video"></i>
                </span>
                <div class="attachment-details">
                    <span class="attachment-name">
                        {{ displayName }}
                    </span>
                    <span v-if="attachment?.extension" class="attachment-extension">.{{ attachment.extension }}</span>
                </div>
            </div>
            <button
                v-if="attachmentUrl && !isLoading"
                :id="`attachmentPreviewViewLink-${attachmentId}`"
                type="button"
                class="view-button"
                @click.prevent.stop="openMediaModal(attachmentUrl, 'video', $event)">
                View Video
            </button>
            <button
                v-else
                :id="`attachmentPreviewViewBtn-${attachmentId}`"
                type="button"
                class="view-button"
                :disabled="isLoading"
                @click.prevent.stop="handleViewClick"
                @mousedown.prevent.stop
                @mouseup.prevent.stop
                @touchstart.prevent.stop
                @touchend.prevent.stop>
                {{ isLoading ? t("common.loading") : t("tickets.detailPage.attachment.preview") }}
            </button>
        </div>

        <div v-else class="attachment-file">
            <div class="attachment-info">
                <span class="attachment-icon" aria-hidden="true">
                    <i class="pi pi-paperclip"></i>
                </span>
                <div class="attachment-details">
                    <span class="attachment-name">
                        {{ displayName }}
                    </span>
                    <span v-if="attachment?.extension" class="attachment-extension">.{{ attachment.extension }}</span>
                </div>
            </div>
            <a v-if="attachmentUrl && !isLoading" :id="`attachmentPreviewViewLink-${attachmentId}`" :href="attachmentUrl" target="_blank" rel="noopener noreferrer" class="view-button" @click.stop>
                View
            </a>
            <button
                v-else
                :id="`attachmentPreviewViewBtn-${attachmentId}`"
                type="button"
                class="view-button"
                :disabled="isLoading"
                @click.prevent.stop="handleViewClick"
                @mousedown.prevent.stop
                @mouseup.prevent.stop
                @touchstart.prevent.stop
                @touchend.prevent.stop>
                {{ isLoading ? t("common.loading") : t("tickets.detailPage.attachment.preview") }}
            </button>
        </div>
    </bx-tile>

    <Dialog v-model:visible="isMediaModalVisible" modal dismissableMask :header="selectedMediaType === 'image' ? 'Image Preview' : 'Video Preview'" :style="{ width: '50vw' }" :breakpoints="{ '1199px': '75vw', '575px': '90vw' }">
        <div class="modal-media-container">
            <img v-if="selectedMediaType === 'image'" :src="selectedMediaUrl" alt="Preview" class="modal-media" />
            <video v-else-if="selectedMediaType === 'video'" :src="selectedMediaUrl" controls autoplay class="modal-media"></video>
        </div>
    </Dialog>
</template>

<style scoped>
.attachment-preview {
    padding: 0.5rem;
    min-width: 200px;
    max-width: 100%;
    position: relative;
}

.attachment-preview ::deep bx-tile {
    position: relative;
}

.media-thumbnail-container {
    width: 2.5rem;
    height: 2.5rem;
    flex-shrink: 0;
    border-radius: 4px;
    overflow: hidden;
    position: relative;
    cursor: pointer;
    background-color: var(--cds-layer-01, #f4f4f4);
    display: flex;
    align-items: center;
    justify-content: center;
    transition: opacity 0.2s ease;
}

.media-thumbnail-container:hover {
    opacity: 0.8;
}

.media-thumbnail {
    width: 100%;
    height: 100%;
    object-fit: cover;
}

.play-overlay {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
    background-color: rgba(0, 0, 0, 0.3);
    color: white;
}

.play-overlay i {
    font-size: 1rem;
}

.modal-media-container {
    display: flex;
    justify-content: center;
    align-items: center;
    padding: 1rem;
    background-color: #f9fafb;
    border-radius: 8px;
}

.modal-media {
    max-width: 100%;
    max-height: 70vh;
    object-fit: contain;
    border-radius: 4px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.attachment-file {
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
    padding: 0.5rem 0;
    position: relative;
    z-index: 1;
}

.attachment-info {
    display: flex;
    align-items: center;
    gap: 0.75rem;
    flex-wrap: wrap;
}

.attachment-icon {
    display: inline-flex;
    align-items: center;
    flex-shrink: 0;
}

.attachment-icon i {
    font-size: 1rem;
}

.attachment-details {
    display: flex;
    align-items: center;
    gap: 0.25rem;
    flex-wrap: wrap;
    flex: 1;
    min-width: 0;
}

.attachment-name {
    font-weight: 500;
    color: var(--cds-text-primary);
    font-size: 0.875rem;
}

.attachment-extension {
    color: var(--cds-text-secondary);
    font-size: 0.875rem;
}

.preview-link {
    text-decoration: none;
    align-self: flex-start;
}

.preview-link:hover {
    text-decoration: none;
}

.view-button {
    padding: 0.5rem 1rem;
    font-size: 0.875rem;
    font-weight: 400;
    background-color: transparent;
    border: 1px solid var(--cds-border-strong);
    color: var(--cds-text-primary);
    cursor: pointer;
    border-radius: 4px;
    transition: all 0.11s cubic-bezier(0.2, 0, 0.38, 0.9);
    align-self: flex-start;
    position: relative;
    z-index: 10;
    pointer-events: auto;
    text-decoration: none;
    display: inline-block;
    text-align: center;
}

.view-button:hover {
    background-color: var(--cds-layer-hover-01);
    border-color: var(--cds-border-strong);
}

.view-button:active {
    background-color: var(--cds-layer-active-01);
}

.view-button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
    pointer-events: none;
}

.view-button:not(:disabled) {
    pointer-events: auto;
    cursor: pointer;
}

.view-button:not(:disabled):hover {
    background-color: var(--cds-layer-hover-01);
    border-color: var(--cds-border-strong);
}

.view-button:not(:disabled):active {
    background-color: var(--cds-layer-active-01);
}

@media (max-width: 768px) {
    .attachment-preview {
        flex-direction: column;
        align-items: flex-start;
    }

    .attachment-info {
        width: 100%;
    }

    .attachment-actions {
        width: 100%;
        justify-content: flex-start;
    }

    .view-button {
        width: 100%;
    }
}
</style>
