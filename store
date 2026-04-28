import { isPlatformBrowser } from '@angular/common';
import {
  ChangeDetectionStrategy,
  Component,
  computed,
  effect,
  ElementRef,
  inject,
  OnDestroy,
  OnInit,
  PLATFORM_ID,
  signal,
  ViewChild,
} from '@angular/core';
import { MatCardModule } from '@angular/material/card';
import Graphic from '@arcgis/core/Graphic';
import { watch } from '@arcgis/core/core/reactiveUtils';
import Point from '@arcgis/core/geometry/Point';
import GraphicsLayer from '@arcgis/core/layers/GraphicsLayer';
import SimpleMarkerSymbol from '@arcgis/core/symbols/SimpleMarkerSymbol';
import TextSymbol from '@arcgis/core/symbols/TextSymbol';
import WebMap from '@arcgis/core/WebMap';
import MapView from '@arcgis/core/views/MapView';

import { DrillingDataStore } from '@store/drilling-data/drilling-data.store';
import { LoaderService } from '@shared/components/global-loader/loader.service';
import { EsriMapService } from '@core/services/esri-map.service';
import { MAP_CONFIG, MAX_WIDTH } from 'src/app/shared/models/config/agwa-map.config';

const BOOT_TASK = 'arcgis-map';
const SELECTED_WELL_ZOOM = 10;

interface SelectedWellCoords {
  readonly lat: number;
  readonly lng: number;
}

@Component({
  selector: 'app-active-wwell-map',
  standalone: true,
  imports: [MatCardModule],
  templateUrl: './active-wwell-map.component.html',
  styleUrl: './active-wwell-map.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class ActiveWwellMapComponent implements OnInit, OnDestroy {
  @ViewChild('presMapViewNode', { static: true }) private mapViewEl?: ElementRef<HTMLDivElement>;

  private readonly platformId = inject(PLATFORM_ID);
  private readonly store = inject(DrillingDataStore);
  private readonly loader = inject(LoaderService);
  private readonly esriAuth = inject(EsriMapService);

  private mapView?: __esri.MapView;
  private selectedWellLayer?: GraphicsLayer;
  private bootTaskRegistered = false;
  private pointerMoveHandle: { remove(): void } | null = null;
  private tooltipEl: HTMLDivElement | null = null;
  private hitTestPending = false;

  protected readonly errorMessage = signal<string | null>('Select a well to preview its location.');
  protected readonly mapReady = signal(false);

  private readonly selectedWellCoords = computed<SelectedWellCoords | null>(() => {
    const header = this.store.wellHeaderData();
    const epANum = this.store.selectedEpANum();

    if (!header || epANum === null) return null;

    const lat = this.parseCoord(header.lat);
    const lng = this.parseCoord(header.lon);

    return lat !== null && lng !== null ? { lat, lng } : null;
  });

  constructor() {
    effect(() => {
      const coords = this.selectedWellCoords();
      if (!this.mapReady()) return;
      this.syncGlowGraphic(coords);
    });
  }

  ngOnInit(): void {
    if (!isPlatformBrowser(this.platformId)) return;
    this.registerBootTask();
    void this.initMap();
  }

  ngOnDestroy(): void {
    this.pointerMoveHandle?.remove();
    this.tooltipEl?.remove();
    this.tooltipEl = null;
    this.mapView?.destroy();
    this.resolveBootTask();
  }

  async captureMapAsBase64(): Promise<string | undefined> {
    if (!this.mapView) return undefined;
    await this.waitForMapRender();
    return this.getScreenshotBase64();
  }

  async saveMapImage(): Promise<void> {
    const base64 = await this.captureMapAsBase64();
    if (!base64) return;
    const a = document.createElement('a');
    a.href = `data:image/png;base64,${base64}`;
    a.download = 'selected-well-map.png';
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
  }

  private parseCoord(value: number | string | null | undefined): number | null {
    const n = typeof value === 'number' ? value : Number.parseFloat(String(value ?? '').trim());
    return Number.isFinite(n) ? n : null;
  }

  private async initMap(): Promise<void> {
    try {
      await this.esriAuth.authenticateUserForMapAccess();

      const webMap = new WebMap({
        portalItem: { id: 'c177620bd7744a22b6e7e0d98c33d18a' },
      });

      this.mapView = new MapView({
        container: this.mapViewEl?.nativeElement,
        map: webMap,
        center: MAP_CONFIG.center,
        zoom: MAP_CONFIG.zoom,
        ui: { components: [] },
      });

      await this.mapView.when();

      this.selectedWellLayer = new GraphicsLayer({ id: 'selected-well' });
      webMap.add(this.selectedWellLayer);

      this.mapReady.set(true);
      this.registerHoverTooltip();
      await this.waitForMapRender();
      this.syncGlowGraphic(this.selectedWellCoords());
      this.resolveBootTask();
    } catch (error) {
      console.error('[ActiveWwellMap] initMap error', error);
      this.errorMessage.set('Map could not be initialized.');
      this.resolveBootTask();
    }
  }

  private syncGlowGraphic(coords: SelectedWellCoords | null): void {
    if (!this.selectedWellLayer) return;

    this.selectedWellLayer.removeAll();

    if (this.store.selectedEpANum() === null) {
      this.errorMessage.set('Select a well to preview its location.');
      return;
    }

    if (!coords) {
      this.errorMessage.set('Selected well does not have valid coordinates.');
      return;
    }

    this.errorMessage.set(null);

    const point = new Point({ latitude: coords.lat, longitude: coords.lng });

    const outerGlow = new Graphic({
      geometry: point,
      symbol: new SimpleMarkerSymbol({
        style: 'circle',
        size: 36,
        color: [14, 165, 233, 0.08],
        outline: { color: [14, 165, 233, 0.35], width: 2 },
      }),
    });

    const midRing = new Graphic({
      geometry: point,
      symbol: new SimpleMarkerSymbol({
        style: 'circle',
        size: 20,
        color: [14, 165, 233, 0.2],
        outline: { color: [14, 165, 233, 0.75], width: 1.5 },
      }),
    });

    const header = this.store.wellHeaderData();
    const core = new Graphic({
      geometry: point,
      symbol: new SimpleMarkerSymbol({
        style: 'circle',
        size: 8,
        color: [14, 165, 233, 1],
        outline: { color: [255, 255, 255, 0.9], width: 1.5 },
      }),
      popupTemplate: header ? {
        title: header.wellName,
        content: `Field: ${header.field} &nbsp;·&nbsp; Depth: ${header.depth} &nbsp;·&nbsp; Ep#: ${header.epNum}<br>Lat: ${coords.lat.toFixed(5)} &nbsp; Lon: ${coords.lng.toFixed(5)}`,
      } : undefined,
    });

    const graphics = [outerGlow, midRing, core];
    const wellName = header?.wellName;

    if (wellName) {
      const label = new Graphic({
        geometry: point,
        symbol: new TextSymbol({
          text: wellName,
          color: [15, 23, 42, 1],
          haloColor: [255, 255, 255, 0.9],
          haloSize: 2,
          font: { size: 11, weight: 'bold', family: 'sans-serif' },
          yoffset: 14,
          horizontalAlignment: 'center',
        }),
      });
      graphics.push(label);
    }

    this.selectedWellLayer.addMany(graphics);
    void this.focusWell(point);
  }

  private registerHoverTooltip(): void {
    if (!this.mapView || !this.mapViewEl) return;

    const tip = document.createElement('div');
    tip.className = 'wwell-map-tooltip';
    this.mapViewEl.nativeElement.appendChild(tip);
    this.tooltipEl = tip;

    this.mapViewEl.nativeElement.addEventListener('mouseleave', () => {
      this.tooltipEl?.classList.remove('wwell-map-tooltip--visible');
    });

    this.pointerMoveHandle = this.mapView.on('pointer-move', (event) => {
      void this.onPointerMove(event);
    });
  }

  private async onPointerMove(event: __esri.ViewPointerMoveEvent): Promise<void> {
    if (!this.mapView || !this.selectedWellLayer || !this.tooltipEl || this.hitTestPending) return;

    this.hitTestPending = true;
    try {
      const result = await this.mapView.hitTest(event, { include: [this.selectedWellLayer] });
      const hit = result.results.some(r => r.type === 'graphic');

      if (hit) {
        this.tooltipEl.textContent = this.store.wellHeaderData()?.wellName ?? '';
        this.tooltipEl.style.left = `${event.x + 14}px`;
        this.tooltipEl.style.top = `${event.y - 10}px`;
        this.tooltipEl.classList.add('wwell-map-tooltip--visible');
      } else {
        this.tooltipEl.classList.remove('wwell-map-tooltip--visible');
      }
    } finally {
      this.hitTestPending = false;
    }
  }

  private async focusWell(point: Point): Promise<void> {
    if (!this.mapView) return;
    try {
      await this.mapView.goTo({ target: point, zoom: SELECTED_WELL_ZOOM }, { animate: false });
    } catch {
      // ignore goTo interruptions during teardown
    }
  }

  private async waitForMapRender(): Promise<void> {
    if (!this.mapView) return;
    if (!this.mapView.updating) {
      await new Promise(requestAnimationFrame);
      return;
    }
    await new Promise<void>((resolve) => {
      const handle = watch(
        () => this.mapView?.updating ?? false,
        (updating) => {
          if (!updating) {
            handle.remove();
            requestAnimationFrame(() => resolve());
          }
        },
      );
    });
  }

  private async getScreenshotBase64(): Promise<string> {
    const screenshot = await this.mapView!.takeScreenshot({ format: 'png', quality: 1 });
    const [, raw] = screenshot.dataUrl.split(',');
    if (!raw) throw new Error('Screenshot payload empty');

    const img = new Image();
    img.src = `data:image/png;base64,${raw}`;
    await new Promise((resolve, reject) => { img.onload = resolve; img.onerror = reject; });

    const scale = Math.min(1, MAX_WIDTH / img.width);
    const canvas = document.createElement('canvas');
    canvas.width = img.width * scale;
    canvas.height = img.height * scale;
    canvas.getContext('2d')!.drawImage(img, 0, 0, canvas.width, canvas.height);

    return canvas.toDataURL('image/jpeg', 1).split(',')[1];
  }

  private registerBootTask(): void {
    if (this.bootTaskRegistered) return;
    this.bootTaskRegistered = true;
    this.loader.registerBootTask(BOOT_TASK);
  }

  private resolveBootTask(): void {
    if (!this.bootTaskRegistered) return;
    this.bootTaskRegistered = false;
    this.loader.resolveBootTask(BOOT_TASK);
  }
}
