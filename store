import { ChangeDetectionStrategy, Component, DestroyRef, effect, inject, OnInit, signal, TemplateRef, ViewChild } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { skip } from 'rxjs';
import { MatDialog, MatDialogModule, MatDialogRef } from '@angular/material/dialog';
import { ActivatedRoute, ParamMap, Router } from '@angular/router';
import { PresentationStore } from './store/presentation.store';
import { formatDateForInput, getTodayAtMidnight, parseDateFromInput } from 'src/app/shared/utils/date.util';
import { ResizeDividerComponent } from '@shared/components/resize-divider/resize-divider.component';
import { WellBoreViewComponent } from './well-bore-view/well-bore-view.component';
import { DepthScaleComponent } from './depth-scale/depth-scale.component';
import { WellNameChipsComponent } from './well-name-chips/well-name-chips.component';
import { MiscPresWellDataComponent } from './misc-pres-well-data/misc-pres-well-data.component';
import { PickedFormationTopsComponent } from './picked-formation-tops/picked-formation-tops.component';
import { ActiveWwellMapComponent } from './active-wwell-map/active-wwell-map.component';
import { OffsetWwellsComponent } from './offset-wwells/offset-wwells.component';
import { WwellsLogsIndicatorsComponent } from './wwells-logs-indicators/wwells-logs-indicators.component';
import { WwellTestResultComponent } from './wwell-test-result/wwell-test-result.component';
import { ActiveWwellDocsViewerComponent } from './active-wwell-docs-viewer/active-wwell-docs-viewer.component';

const LEFT_DEFAULT = 320; const LEFT_MIN = 220; const LEFT_MAX = 520;
const RIGHT_DEFAULT = 340; const RIGHT_MIN = 240; const RIGHT_MAX = 560;

@Component({
  selector: 'app-presentation',
  standalone: true,
  imports: [
    MatDialogModule,
    ResizeDividerComponent,
    WellBoreViewComponent,
    DepthScaleComponent,
    WellNameChipsComponent,
    MiscPresWellDataComponent,
    PickedFormationTopsComponent,
    ActiveWwellMapComponent,
    OffsetWwellsComponent,
    WwellsLogsIndicatorsComponent,
    WwellTestResultComponent,
    ActiveWwellDocsViewerComponent,
  ],
  templateUrl: './presentation.component.html',
  styleUrl: './presentation.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class PresentationComponent implements OnInit {
  @ViewChild('freeLayoutDialog') private freeLayoutDialog?: TemplateRef<unknown>;

  protected readonly store = inject(PresentationStore);
  private readonly destroyRef = inject(DestroyRef);
  private readonly route = inject(ActivatedRoute);
  private readonly router = inject(Router);
  private readonly dialog = inject(MatDialog);
  private dialogRef?: MatDialogRef<unknown>;

  /** false = fixed three-column (default), true = free draggable modal */
  protected readonly draggableMode = signal(false);
  protected readonly loadingVisible = signal(false);

  protected readonly leftWidth = signal(LEFT_DEFAULT);
  protected readonly rightWidth = signal(RIGHT_DEFAULT);

  private loadingTimer: ReturnType<typeof setTimeout> | null = null;
  private loadingShownAt = 0;

  constructor() {
    effect(() => {
      const isLoading = this.store.isDetailsLoading();

      if (isLoading) {
        this.clearLoadingTimer();
        this.loadingShownAt = Date.now();
        this.loadingVisible.set(true);
        return;
      }

      if (!this.loadingVisible()) {
        return;
      }

      const elapsed = Date.now() - this.loadingShownAt;
      const remaining = Math.max(0, 380 - elapsed);

      this.clearLoadingTimer();
      this.loadingTimer = setTimeout(() => {
        this.loadingVisible.set(false);
        this.loadingTimer = null;
      }, remaining);
    });

    // Sync store state back to URL. Must live in the constructor so it is registered
    // before ngOnInit runs applyQueryParams — otherwise a selectedDate change from the
    // tap (which fires synchronously inside initialize) can trigger this effect with a
    // stale selectedEpANum, overwriting the URL before the HTTP response arrives.
    effect(() => {
      const epANum = this.store.selectedEpANum();
      const date = this.store.selectedDate();
      if (epANum == null) return;
      this.router.navigate([], {
        relativeTo: this.route,
        queryParams: { epANum, date: formatDateForInput(date) },
        queryParamsHandling: 'merge',
        replaceUrl: true,
      });
    });

    this.destroyRef.onDestroy(() => this.clearLoadingTimer());
  }

  ngOnInit(): void {
    this.applyQueryParams(this.route.snapshot.queryParamMap, true);

    this.route.queryParamMap
      .pipe(skip(1), takeUntilDestroyed(this.destroyRef))
      .subscribe((params) => this.applyQueryParams(params));
  }

  protected openDraggableDialog(): void {
    if (!this.freeLayoutDialog || this.draggableMode()) return;

    this.draggableMode.set(true);
    this.dialogRef = this.dialog.open(this.freeLayoutDialog, {
      autoFocus: false,
      panelClass: 'pres-modal-dialog',
      width: '92vw',
      maxWidth: '92vw',
      height: '90vh',
    });

    this.dialogRef.afterClosed().subscribe(() => {
      this.draggableMode.set(false);
      this.dialogRef = undefined;
    });
  }

  protected closeDraggableDialog(): void {
    this.dialogRef?.close();
  }

  protected onLeftDrag(delta: number): void {
    this.leftWidth.update(w => Math.min(LEFT_MAX, Math.max(LEFT_MIN, w + delta)));
  }

  protected onRightDrag(delta: number): void {
    this.rightWidth.update(w => Math.min(RIGHT_MAX, Math.max(RIGHT_MIN, w - delta)));
  }

  private clearLoadingTimer(): void {
    if (this.loadingTimer !== null) {
      clearTimeout(this.loadingTimer);
      this.loadingTimer = null;
    }
  }

  private applyQueryParams(params: ParamMap, forceLoad = false): void {
    const requestedDate = this.normalizeDateParam(params.get('date'));
    const rawEpANum = params.get('epANum');
    const epANum = rawEpANum ? Number.parseInt(rawEpANum, 10) : null;
    const currentDate = formatDateForInput(this.store.selectedDate());
    const currentEpANum = this.store.selectedEpANum();

    if (!forceLoad && requestedDate === currentDate && (epANum ?? null) === currentEpANum) {
      return;
    }

    this.store.initialize(
      requestedDate,
      epANum != null && !Number.isNaN(epANum) ? epANum : undefined,
    );
  }

  private normalizeDateParam(value: string | null): string {
    if (!value) {
      return formatDateForInput(getTodayAtMidnight());
    }

    const parsed = parseDateFromInput(value);
    const today = getTodayAtMidnight();

    if (Number.isNaN(parsed.getTime()) || parsed > today) {
      return formatDateForInput(today);
    }

    return value;
  }
}
