import {
  ChangeDetectionStrategy,
  Component,
  ElementRef,
  effect,
  inject,
  signal,
} from '@angular/core';
import { CommonModule } from '@angular/common';
import { MatDialog } from '@angular/material/dialog';
import { MatProgressSpinnerModule } from '@angular/material/progress-spinner';
import { MatIconModule } from '@angular/material/icon';
import { animate, query, stagger, style, transition, trigger } from '@angular/animations';
import { DrillingDataStore } from '@store/drilling-data/drilling-data.store';
import { WellDocsStore } from '@store/well-docs/well-docs.store';
import { PresDocsService } from '@services/pres-docs.service';
import { FilePreviewDialogComponent } from '@shared/components/file-upload/file-preview-dialog/file-preview-dialog.component';
import { formatDateForInput } from 'src/app/shared/utils/date.util';

@Component({
  selector: 'app-active-wwell-docs-viewer',
  standalone: true,
  imports: [CommonModule, MatProgressSpinnerModule, MatIconModule],
  templateUrl: './active-wwell-docs-viewer.component.html',
  styleUrl: './active-wwell-docs-viewer.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush,
  animations: [
    trigger('expandCollapse', [
      transition(':enter', [
        style({ height: 0, opacity: 0, overflow: 'hidden' }),
        animate('280ms cubic-bezier(0.4, 0, 0.2, 1)', style({ height: '*', opacity: 1 })),
      ]),
      transition(':leave', [
        style({ height: '*', opacity: 1, overflow: 'hidden' }),
        animate('220ms cubic-bezier(0.4, 0, 0.2, 1)', style({ height: 0, opacity: 0 })),
      ]),
    ]),
    trigger('listStagger', [
      transition(':enter', [
        query('.doc-row', [
          style({ opacity: 0, transform: 'translateX(-10px)' }),
          stagger(60, animate('240ms ease', style({ opacity: 1, transform: 'translateX(0)' }))),
        ], { optional: true }),
      ]),
    ]),
  ],
})
export class ActiveWwellDocsViewerComponent {
  protected readonly wellStore = inject(DrillingDataStore);
  protected readonly docsStore = inject(WellDocsStore);
  private readonly svc = inject(PresDocsService);
  private readonly dialog = inject(MatDialog);
  private readonly el = inject(ElementRef<HTMLElement>);

  protected readonly collapsed = signal(true);
  protected readonly viewLoading = signal<Set<string>>(new Set());
  protected readonly downloadLoading = signal<Set<string>>(new Set());

  constructor() {
    effect(() => {
      const epANum = this.wellStore.selectedEpANum();
      const date = formatDateForInput(this.wellStore.selectedDate());
      if (epANum == null) return;
      this.docsStore.loadDocList({ epANum, date });
    });
  }

  protected toggleCollapse(): void {
    this.collapsed.update(v => {
      if (v) setTimeout(() => this.scrollToSelf(), 300);
      return !v;
    });
  }

  protected viewDoc(docName: string): void {
    const context = this.getContext();
    if (!context || this.isViewLoading(docName)) return;

    this.setLoading(this.viewLoading, docName, true);
    this.svc.fetchDoc(docName, context.epANum, context.date).subscribe({
      next: (blob) => {
        const mime = blob.type && blob.type !== 'application/octet-stream'
          ? blob.type
          : this.guessMime(docName);
        const file = new File([blob], docName, { type: mime });
        this.setLoading(this.viewLoading, docName, false);
        this.dialog.open(FilePreviewDialogComponent, {
          data: file,
          width: '60vw',
          height: '70vh',
          maxWidth: '98vw',
          maxHeight: '98vh',
          panelClass: 'file-preview-panel',
          autoFocus: false,
          enterAnimationDuration: '220ms',
        });
      },
      error: () => this.setLoading(this.viewLoading, docName, false),
    });
  }

  protected downloadDoc(docName: string): void {
    const context = this.getContext();
    if (!context || this.isDownloadLoading(docName)) return;

    this.setLoading(this.downloadLoading, docName, true);
    this.svc.fetchDoc(docName, context.epANum, context.date).subscribe({
      next: (blob) => {
        const url = URL.createObjectURL(blob);
        const anchor = document.createElement('a');
        anchor.href = url;
        anchor.download = docName;
        document.body.appendChild(anchor);
        anchor.click();
        document.body.removeChild(anchor);
        setTimeout(() => URL.revokeObjectURL(url), 5000);
        this.setLoading(this.downloadLoading, docName, false);
      },
      error: () => this.setLoading(this.downloadLoading, docName, false),
    });
  }

  protected isViewLoading(docName: string): boolean {
    return this.viewLoading().has(docName);
  }

  protected isDownloadLoading(docName: string): boolean {
    return this.downloadLoading().has(docName);
  }

  protected fileIcon(docName: string): string {
    const ext = docName.split('.').pop()?.toLowerCase() ?? '';
    if (ext === 'pdf') return 'picture_as_pdf';
    if (['jpg', 'jpeg', 'png', 'gif', 'webp'].includes(ext)) return 'image';
    if (['xls', 'xlsx'].includes(ext)) return 'table_chart';
    if (['doc', 'docx'].includes(ext)) return 'article';
    return 'insert_drive_file';
  }

  protected fileTypeClass(docName: string): string {
    const ext = docName.split('.').pop()?.toLowerCase() ?? '';
    if (ext === 'pdf') return 'pdf';
    if (['jpg', 'jpeg', 'png', 'gif', 'webp'].includes(ext)) return 'image';
    if (['xls', 'xlsx'].includes(ext)) return 'excel';
    if (['doc', 'docx'].includes(ext)) return 'word';
    return 'other';
  }

  private getContext(): { epANum: number; date: string } | null {
    const epANum = this.wellStore.selectedEpANum();
    if (epANum == null) return null;
    return { epANum, date: formatDateForInput(this.wellStore.selectedDate()) };
  }

  private setLoading(target: ReturnType<typeof signal<Set<string>>>, docName: string, loading: boolean): void {
    target.update((set: Set<string>) => {
      const next = new Set(set);
      loading ? next.add(docName) : next.delete(docName);
      return next;
    });
  }

  private guessMime(docName: string): string {
    const ext = docName.split('.').pop()?.toLowerCase() ?? '';
    const map: Record<string, string> = {
      pdf: 'application/pdf',
      jpg: 'image/jpeg', jpeg: 'image/jpeg', png: 'image/png',
      gif: 'image/gif', webp: 'image/webp',
      doc: 'application/msword',
      docx: 'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
      xls: 'application/vnd.ms-excel',
      xlsx: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
    };
    return map[ext] ?? 'application/octet-stream';
  }

  private scrollToSelf(): void {
    const el: HTMLElement = this.el.nativeElement;
    const scrollParent = this.findScrollParent(el);
    if (!scrollParent) return;
    const offset = el.getBoundingClientRect().top - scrollParent.getBoundingClientRect().top;
    scrollParent.scrollTo({ top: scrollParent.scrollTop + offset, behavior: 'smooth' });
  }

  private findScrollParent(el: HTMLElement): HTMLElement | null {
    let parent = el.parentElement;
    while (parent && parent !== document.body) {
      const { overflowY } = window.getComputedStyle(parent);
      if (overflowY === 'auto' || overflowY === 'scroll') return parent;
      parent = parent.parentElement;
    }
    return null;
  }
}
