import { computed, inject } from '@angular/core';
import { patchState, signalStore, withComputed, withMethods, withState } from '@ngrx/signals';
import { rxMethod } from '@ngrx/signals/rxjs-interop';
import { tapResponse } from '@ngrx/operators';
import { pipe, switchMap, tap } from 'rxjs';

import { WellDoc, WellDocsState } from '@models/well-design/well-docs.model';
import { DocListResponse, UploadDocResponse } from '@shared/models/wwell/api-response.model';
import { PresDocsService } from '@services/pres-docs.service';
import { NotificationService } from '@shared/components/notification/notification.service';
import { DUMMY_DOCS, initialWellDocsState } from './well-docs.state';
import { selectCategories, selectDocsByCategory, selectTotalDocCount } from './well-docs.selectors';
import { WellDocsActions, WellDocsEvents, LoadDocsPayload, UploadDocsPayload } from './well-docs.actions';

export const WellDocsStore = signalStore(
  { providedIn: 'root' },

  withState<WellDocsState>(initialWellDocsState),

  withComputed(({ docs }) => ({
    docsByCategory: computed(() => selectDocsByCategory(docs())),
    categories:     computed(() => selectCategories(docs())),
    totalCount:     computed(() => selectTotalDocCount(docs())),
  })),

  withMethods((store, svc = inject(PresDocsService), notify = inject(NotificationService)) => ({
    loadDocs: rxMethod<LoadDocsPayload>(
      pipe(
        tap(() => patchState(store, { loading: true, error: null })),
        switchMap(({ epANum, date }) =>
          svc.getDocs(epANum, date).pipe(
            tapResponse({
              next: (docs: WellDoc[]) =>
                patchState(store, {
                  docs: docs.length > 0 ? docs : DUMMY_DOCS,
                  loading: false,
                }),
              error: (err: unknown) => {
                const msg = err instanceof Error ? err.message : 'Failed to load documents';
                patchState(store, { docs: DUMMY_DOCS, loading: false, error: msg });
                notify.error(msg);
              },
            }),
          ),
        ),
      ),
    ),

    uploadDocs: rxMethod<UploadDocsPayload>(
      pipe(
        tap(() => patchState(store, { uploading: true, error: null })),
        switchMap(({ files, epANum, date }) =>
          svc.uploadDocs(files, epANum, date).pipe(
            tapResponse({
              next: (responses: UploadDocResponse[]) => {
                const failed = responses.filter(r => r.error);
                if (failed.length) {
                  const msg = failed.map(r => r.message).join('; ');
                  patchState(store, { uploading: false, error: msg });
                  notify.error(msg);
                  return;
                }
                notify.info('Documents uploaded successfully.');
                svc.getDocs(epANum, date).subscribe({
                  next: docs =>
                    patchState(store, { docs: docs.length > 0 ? docs : DUMMY_DOCS, uploading: false }),
                  error: () =>
                    patchState(store, { uploading: false }),
                });
              },
              error: (err: unknown) => {
                const msg = err instanceof Error ? err.message : 'Upload failed';
                patchState(store, { uploading: false, error: msg });
                notify.error(msg);
              },
            }),
          ),
        ),
      ),
    ),

    loadDocList: rxMethod<LoadDocsPayload>(
      pipe(
        tap(() => patchState(store, { listLoading: true, listError: null })),
        switchMap(({ epANum, date }) =>
          svc.getDocList(epANum, date).pipe(
            tapResponse({
              next: (res: DocListResponse) => {
                if (res.error) {
                  patchState(store, { docNames: [], listLoading: false, listError: res.message ?? 'Failed to load documents' });
                  notify.error(res.message ?? 'Failed to load documents');
                  return;
                }
                patchState(store, { docNames: res.data.totalFiles, listLoading: false });
              },
              error: (err: unknown) => {
                const msg = err instanceof Error ? err.message : 'Failed to load documents';
                patchState(store, { docNames: [], listLoading: false, listError: msg });
                notify.error(msg);
              },
            }),
          ),
        ),
      ),
    ),

    openViewer(doc: WellDoc): void {
      patchState(store, { selectedDoc: doc, viewerOpen: true });
    },

    closeViewer(): void {
      patchState(store, { selectedDoc: null, viewerOpen: false });
    },

    clearDocs(): void {
      patchState(store, initialWellDocsState);
    },
  })),
);

export { WellDocsActions, WellDocsEvents };
