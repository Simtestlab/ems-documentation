# Application Architecture 

## Folder Structure

The application’s folder structure is modeled, prioritizing streamlined code organization, reduced redundancy, and a clearly defined, easily navigable project architecture.

## Folder Structure

```text
src/
├── application/
│   └── services/
│       ├── metrics.service.ts
│       └── space.service.ts
├── auth/
│   ├── AuthContext.tsx
│   ├── permissions.ts
│   ├── ProtectedFeature.tsx
│   └── usePermissions.ts
├── domain/
│   ├── entities/
│   │   ├── alert.entity.ts
│   │   ├── analytics.entity.ts
│   │   ├── equipment.entity.ts
│   │   ├── hierarchy.entity.ts
│   │   ├── metrics.entity.ts
│   │   ├── space.entity.ts
│   │   ├── time-series.entity.ts
│   │   └── weather.entity.ts
│   └── interfaces/
│       └── repositories.interface.ts
├── infrastructure/
│   ├── api/
│   │   ├── metrics.repository.ts
│   │   └── space.repository.ts
│   ├── ems/
│   │   ├── ems-engine.ts
│   │   └── ems-state.ts
│   ├── mock/
│   │   ├── alerts.mock.ts
│   │   ├── analytics.mock.ts
│   │   ├── equipment.mock.ts
│   │   ├── hierarchy-data.ts
│   │   ├── metrics.mock.ts
│   │   ├── metrics.mocks.old
│   │   ├── multi-site-hierarchy.ts
│   │   ├── space.mock.ts
│   │   └── weather.mock.ts
│   ├── simulation/
│   │   ├── aggregation-engine.ts
│   │   ├── battery-controller.ts
│   │   ├── hierarchy-manager.ts
│   │   ├── load-simulator.ts
│   │   ├── simulation-config.ts
│   │   ├── solar-simulator.ts
│   │   ├── space-simulator.ts
│   │   └── weather-simulator.ts
│   └── time/
│       ├── time-engine.ts
│       └── time-series-store.ts
├── presentation/
│   ├── components/
│   │   ├── features/
│   │   │   ├── map/
│   │   │   │   ├── utils/
│   │   │   │   │   └── markerUtils.ts
│   │   │   │   ├── index.ts
│   │   │   │   ├── MapLegend.tsx
│   │   │   │   └── SiteDetailPanel.tsx
│   │   │   ├── AlertsPanel.tsx
│   │   │   ├── BackendConnectionError.tsx
│   │   │   ├── DashboardFooter.tsx
│   │   │   ├── DashboardHeader.tsx
│   │   │   ├── EnergyAnalyticsPanel.tsx
│   │   │   ├── EnergyFlowDiagram.tsx
│   │   │   ├── RealtimeChart.tsx
│   │   │   ├── SiteComparisonCard.tsx
│   │   │   ├── SitesMap.tsx
│   │   │   ├── SiteWiseEnergyFlow.tsx
│   │   │   ├── SummaryCard.tsx
│   │   │   ├── TariffAnalysisPanel.tsx
│   │   │   ├── TimeControlPanel.tsx
│   │   │   └── WeatherWidget.tsx
│   │   └── ui/
│   │       └── SparklineChart.tsx
│   └── hooks/
│       ├── useAggregatedMetrics.ts
│       ├── useAlerts.api.ts
│       ├── useAnalytics.api.ts
│       ├── useBackendHealth.ts
│       ├── useBusinessMetrics.api.ts
│       ├── useBusinessMetrics.ts
│       ├── useEnergyMetrics.api.ts
│       ├── useEnergyMetrics.ts
│       ├── useLiveTelemetry.api.ts
│       ├── useLiveTelemetry.ts
│       ├── useRealtimeMetrics.ts
│       ├── useRealtimeSiteChartData.api.ts
│       ├── useRealtimeSiteChartData.ts
│       ├── useRealtimeSiteMetrics.api.ts
│       ├── useRealtimeSiteMetrics.ts
│       ├── useTariff.api.ts
│       ├── useTimeEngine.ts
│       ├── useVisualThrottle.ts
│       └── useWeather.api.ts
├── shared/
│   ├── config/
│   │   └── api.config.ts
│   ├── constants/
│   │   └── colors.ts
│   └── utils/
│       ├── chart-config.ts
│       ├── cn.ts
│       ├── formatting.ts
│       ├── physics.ts
│       └── site-metrics.ts
└── middleware.ts
```
