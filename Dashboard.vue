<!-- src/views/Dashboard.vue -->
<template>
  <div
    class="dashboard-root"
    :class="rootClasses"
    @click="closeAllTransientUI"
    data-testid="dashboard-root"
  >
    <!-- ===== FATAL/CHILD ERROR TOAST (non-blocking) ===== -->
    <div v-if="fatalError" class="app-error" role="alert" aria-live="assertive">
      <strong>Something hiccuped:</strong> {{ fatalError }}
      <button class="btn micro ghost" @click="fatalError = ''">Dismiss</button>
    </div>

    <!-- ========== LEFT SIDEBAR ========== -->
    <aside class="sidebar" :class="{ collapsed: sidebarCollapsed }" @click.stop aria-label="Primary">
      <div class="sidebar-top">
        <div class="brand" @click.stop="openHomeInNewTab" role="link" tabindex="0"
             @keydown.enter.prevent="openHomeInNewTab"
             @keydown.space.prevent="openHomeInNewTab">
          <img class="brand-emblem" :src="wickwiseEmblem" alt="WickWise Emblem" />
          <span class="brand-text" v-if="!sidebarCollapsed">WickWise</span>
        </div>
        <button
          class="collapse-btn"
          @click="sidebarCollapsed = !sidebarCollapsed; persistSidebar()"
          :aria-expanded="!sidebarCollapsed"
          :aria-label="sidebarCollapsed ? 'Expand sidebar' : 'Collapse sidebar'"
          data-testid="btn-collapse-sidebar"
        >
          {{ sidebarCollapsed ? '›' : '‹' }}
        </button>
      </div>

      <div v-if="!sidebarCollapsed" class="sidebar-scroll">
        <div class="side-tabs" role="navigation" aria-label="Sidebar">
          <!-- Chart type (dropdown in tab) -->
          <div class="tab-with-menu">
            <button
              :class="{ active: chartMenuOpen }"
              class="side-tab-btn"
              @click.stop="toggleChartTypeMenu"
              aria-haspopup="listbox"
              :aria-expanded="chartMenuOpen ? 'true' : 'false'"
              aria-controls="chart-type-menu"
              title="Select chart type"
              data-testid="btn-chart-type"
            >
              📊 <span>Chart type</span> <span class="caret">▾</span>
            </button>
            <div
              v-if="chartMenuOpen"
              id="chart-type-menu"
              class="tab-menu"
              role="listbox"
              @click.stop
            >
              <button
                class="tab-menu-item"
                role="option"
                :aria-selected="ui.chartMode==='candlestick'"
                :class="{ sel: ui.chartMode==='candlestick' }"
                @click="setChartMode('candlestick')"
                data-testid="opt-chart-candles"
              >
                Candlestick
              </button>
              <button
                class="tab-menu-item"
                role="option"
                :aria-selected="ui.chartMode==='line'"
                :class="{ sel: ui.chartMode==='line' }"
                @click="setChartMode('line')"
                data-testid="opt-chart-line"
              >
                Line
              </button>
            </div>
          </div>

          <!-- Markets now opens a DROP PANEL from the header -->
          <button :class="{ active: leftTab === 'markets' }" @click="openMarketsPanel" title="Markets" data-testid="btn-markets"> 🧭 <span>Markets</span> </button>

          <!-- Zeeno button: wakes Zeeno, modifier = new tab -->
          <button :class="{ active: true }" @click="openZeeno" title="Zeeno (Auto-Analyse)" data-testid="btn-zeeno">🧠 <span>Zeeno</span></button>

          <button :class="{ active: false }" @click="instantToggleTheme" title="Toggle theme" data-testid="btn-theme">🌓 <span>Theme</span></button>

          <button :class="{ active: leftTab === 'pairs' }" @click="openOverlay('pairs')" title="Pairs" data-testid="btn-pairs">🔗 <span>Pairs</span></button>

          <!-- Settings (toggles drop-down panel) & Logout -->
          <button :class="{ active: settingsOpen }" @click="toggleSettings" title="Settings" data-testid="btn-settings">⚙️ <span>Settings</span></button>
          <button :class="{ active: false }" @click="logout" title="Logout" data-testid="btn-logout">🚪 <span>Logout</span></button>
        </div>
      </div>
    </aside>

    <!-- ========== MAIN ========== -->
    <main class="main" @click.stop>
      <!-- Optional service notice / empty state -->
      <div v-if="serviceNotice" class="service-notice" role="alert" aria-live="polite" data-testid="service-notice">
        {{ serviceNotice }}
      </div>

      <!-- ===== POWERED BY / PROVIDERS BAR (moved ABOVE top movers) ===== -->
      <div class="powerbar" role="contentinfo" aria-label="Data providers">
        <span class="pb-title">Powered by</span>
        <ul class="pb-list" v-if="providerChips.length">
          <li
            v-for="p in providerChips"
            :key="p.id"
            class="pb-chip"
            :class="{ off: !p.ok }"
            :title="p.title"
          >
            <img class="pb-ic" :src="p.icon" :alt="p.name" @error="onProviderIconError($event, p)" />
            <span class="pb-name">{{ p.name }}</span>
            <span class="pb-dot" :class="p.ok ? 'ok' : 'warn'"></span>
          </li>
        </ul>
        <span class="pb-pill" :class="{ warn: !serverLiveData }">
          {{ serverLiveData ? 'Live data' : 'Fallback mode' }}
        </span>
      </div>

      <!-- ===== SETTINGS DROP PANEL (restored) ===== -->
      <transition name="settings-drop">
        <section
          v-if="settingsOpen"
          ref="settingsEl"
          class="settings-panel"
          role="dialog"
          aria-modal="true"
          aria-label="Settings"
          tabindex="0"
          @click.stop
        >
          <div class="sp-head">
            <h3 class="sp-title">Settings</h3>
            <div class="sp-actions">
              <button class="btn tiny" @click="toggleSettings">Close</button>
            </div>
          </div>
          <div class="sp-body">
            <div style="display:grid; gap:8px;">
              <div><strong>Theme:</strong> <button class="btn tiny" @click="instantToggleTheme">Toggle ({{ theme }})</button></div>
              <div>
                <strong>Timeframe:</strong>
                <select v-model="ui.tf" class="cc-select" data-testid="settings-tf">
                  <option>1m</option><option>5m</option><option>15m</option>
                  <option>1h</option><option>4h</option><option>1d</option>
                </select>
              </div>
              <div><strong>Subscription:</strong> {{ devPlanLabel }}</div>
              <div><strong>User ID:</strong> <code>{{ userId }}</code></div>
            </div>
          </div>
        </section>
      </transition>

      <!-- HEADER (Top Movers chips) -->
      <div class="topbar" @click.stop>
        <div class="mini-movers" aria-live="polite">
          <span class="mini-title">Top Movers:</span>
          <template v-if="headerMovers.length">
            <button
              v-for="m in headerMovers"
              :key="m.asset + ':' + m.symbol"
              class="mini-chip"
              type="button"
              @click="onTopMoverClick($event, m)"
              @keydown.enter.prevent="onTopMoverClick($event, m)"
              @keydown.space.prevent="onTopMoverClick($event, m)"
              @dblclick="openFullChartNewTab({ key:m.id || m.symbol, class:m.asset })"
              :title="${prettyMarket(m.asset)} · ${m.symbol} · ${formatPrice(m.price)} · ${signedPct(m.change)} · Alt-click to pin"
              data-testid="mini-mover"
            >
              <img class="mini-ic" :src="iconFor(m.asset, m.symbol)" :alt="m.symbol" @error="onIconError($event, m.asset)" />
              <span class="mini-market">{{ prettyMarket(m.asset) }}</span>
              <span class="mini-symbol">{{ m.symbol }}</span>
              <span class="mini-chg" :class="m.change >=0 ? 'up':'down'">{{ signedPct(m.change) }}</span>
            </button>
          </template>
          <span v-else class="mini-hint">Loading movers…</span>
          <button class="btn micro ghost" style="margin-left:6px" @click="refreshTop" title="Reload movers" data-testid="btn-reload-movers">Reload</button>
        </div>
      </div>

      <!-- ====== MARKETS DROP PANEL (from header) ====== -->
      <transition name="markets-drop">
        <section
          v-if="marketPanel.open"
          class="markets-panel"
          role="dialog"
          aria-modal="true"
          aria-label="Markets"
          tabindex="0"
          ref="marketEl"
          @click.stop
          data-testid="panel-markets"
          @keydown.esc.stop.prevent="closeMarketsPanel"
        >
          <div class="mp-head">
            <h3 class="mp-title">Markets</h3>
            <div class="mp-actions">
              <div class="tabs small" style="margin-right:8px;">
                <button v-if="tvEnabled" :class="{ active: marketsView==='overview' }" @click="marketsView='overview'" data-testid="tab-overview">Overview</button>
                <button :class="{ active: marketsView==='list' || !tvEnabled }" @click="marketsView='list'" data-testid="tab-list">List</button>
                <!-- NEW: Top 4 tab renders INSIDE the Markets panel -->
                <button :class="{ active: marketsView==='top4' }" @click="openTop4InPanel" data-testid="tab-top4">Top 4</button>
              </div>
              <button class="btn tiny" @click="refreshTop">Reload</button>
              <button class="btn tiny" @click="closeMarketsPanel">Close</button>
            </div>
          </div>

          <div class="mp-body">
            <div v-if="tvEnabled && marketsView==='overview'" class="mp-overview disabled">
              <div class="empty small" style="padding:10px;">Market Overview embeds are disabled.</div>
            </div>

            <div v-else-if="marketsView==='list' || !tvEnabled" class="mp-list scroll-2d">
              <div class="tabs small scroll" v-hscroll style="margin-bottom:8px;">
                <button v-for="t in drawerTabs" :key="t.key" :class="{ active: marketTab === t.key }" @click="marketTab = t.key">{{ t.label }}</button>
              </div>

              <div class="mp-list-inner" :aria-busy="!visibleDrawerItems.length ? 'true' : undefined">
                <div
                  v-for="it in visibleDrawerItems"
                  :key="it.key"
                  class="row"
                  @click="quickSelectFromChip(it)"
                  @dblclick="openFullChartNewTab(it)"
                  :title="${it.label} · ${formatPrice(it.price)} · ${signedPct(it.change)}"
                  data-testid="row-market-item"
                >
                  <img class="row-ic" :src="it.icon || iconFor(it.asset, it.symbol || it.key)" :alt="it.label" @error="onIconError($event, it.asset)" />
                  <span class="row-name">{{ it.label }}</span>
                  <span class="row-price">{{ formatPrice(it.price) }}</span>
                  <span class="row-chg" :class="(it.change ?? 0) >= 0 ? 'up' : 'down'">{{ signedPct(it.change) }}</span>
                  <button
                    class="icon-btn pin"
                    :class="{ active: inWatchlist(it, it.asset) }"
                    @click.stop="toggleWatchlist(it, it.asset)"
                    :title="inWatchlist(it, it.asset) ? 'Unpin' : 'Pin'"
                    aria-label="Pin"
                    :aria-pressed="inWatchlist(it, it.asset) ? 'true' : 'false'"
                    data-testid="btn-pin-market-item"
                  >
                    <svg viewBox="0 0 24 24" class="ic"><path d="M14 2l1 5 3 3-5 1-6 8-1-1 8-6 1-5z"/></svg>
                  </button>
                </div>
                <div class="empty" v-if="!visibleDrawerItems.length">No data yet. Check API keys on the server.</div>
              </div>
            </div>

            <!-- NEW: Top-4 rendered INSIDE Markets panel -->
            <div v-else-if="marketsView==='top4'" class="mp-list scroll-2d" data-testid="panel-top4">
              <div class="top4-row">
                <label class="top4-label">Market</label>
                <select v-model="top4Market" class="top4-select" @change="loadTop4ForMarket" data-testid="select-top4-market">
                  <option v-for="opt in marketOptions" :key="opt.key" :value="opt.key">{{ opt.label }}</option>
                </select>
                <button class="btn tiny" @click="loadTop4ForMarket">Reload</button>
              </div>

              <div class="top4-grid">
                <div
                  v-for="it in top4Visible"
                  :key="it.key"
                  class="top4-item"
                  @click="quickSelectFromChip(it)"
                  :title="${it.label} · C ${formatPrice(it.close)} · H ${formatPrice(it.high)} · L ${formatPrice(it.low)}"
                >
                  <div class="row1">
                    <div class="sym-wrap">
                      <img class="row-ic" :src="iconFor(it.asset, it.symbol)" :alt="it.label" @error="onIconError($event, it.asset)" />
                      <span class="sym">{{ it.label }}</span>
                    </div>
                    <span class="chg" :class="(it.change ?? 0) >= 0 ? 'up' : 'down'">{{ signedPct(it.change) }}</span>
                  </div>
                  <div class="row2">
                    <span class="px">C {{ formatPrice(it.close ?? it.price) }}</span>
                    <span class="px">H {{ formatPrice(it.high) }}</span>
                    <span class="px">L {{ formatPrice(it.low) }}</span>
                  </div>
                  <div class="row3">
                    <button class="btn micro" @click.stop="openFullChartNewTab({ key: it.id || it.symbol, class: it.asset })">Full chart</button>
                    <button
                      class="icon-btn pin"
                      :class="{ active: inWatchlist(it, it.asset) }"
                      @click.stop="pinTop4(it)"
                      :title="inWatchlist(it, it.asset) ? 'Unpin' : 'Pin'"
                      aria-label="Pin"
                      :aria-pressed="inWatchlist(it, it.asset) ? 'true' : 'false'"
                    >
                      <svg viewBox="0 0 24 24" class="ic"><path d="M14 2l1 5 3 3-5 1-6 8-1-1 8-6 1-5z"/></svg>
                    </button>
                  </div>
                </div>
              </div>
            </div>
            <!-- /Top-4 in Markets panel -->
          </div>
        </section>
      </transition>

      <!-- KPI + AI Overview + Actions -->
      <div class="kpi-row">
        <div class="kpi-card">
          <div class="kpi-title">Selected</div>
          <div class="kpi-selected">
            <div class="kpi-value">
              <img
                class="title-ic"
                :src="iconFor(activeClass, activeClass==='crypto' ? (activeName || activeSymbol) : activeSymbol)"
                :alt="selectedLabel"
                @error="onIconError($event, activeClass)"
              />
              <span>{{ selectedLabel }}</span>
            </div>
            <div class="kpi-sub" :class="(changeForSelected ?? 0) >= 0 ? 'up' : 'down'">
              {{ signedPct(changeForSelected) }}
            </div>
          </div>
        </div>

        <div class="kpi-card highlight">
          <div class="kpi-title">AI Overview — {{ selectedLabel }}</div>
          <div class="ai-overview" v-if="aiOverview.ready">
            <div class="ao-headline">{{ aiOverview.title }}</div>
            <div class="ao-tags">
              <span class="tag" :class="aiOverview.momentumClass">{{ aiOverview.momentum }}</span>
              <span class="tag" :class="aiOverview.regimeClass">{{ aiOverview.regime }}</span>
              <span class="tag">{{ aiOverview.volatility }}</span>
              <span class="tag ghost">live</span>
            </div>
            <p class="ao-summary">{{ aiOverview.summary }}</p>
            <ul class="ao-bullets">
              <li v-for="(b, i) in aiOverview.bullets" :key="i">{{ b }}</li>
            </ul>
            <div class="ao-foot"><span class="muted">Zeeno read:</span> {{ aiOverview.zeenoTake }}</div>
          </div>
          <div class="ai-overview empty" v-else>
            <div class="ao-headline">Gathering data…</div>
            <p class="ao-summary">Waiting for enough candles to compile a session recap and live trend read.</p>
          </div>
        </div>

        <div class="kpi-card">
          <div class="kpi-title">Actions</div>
          <div class="kpi-actions spaced" role="group" aria-label="Chart actions">
            <button class="btn tiny" @click="refreshAll" data-testid="btn-refresh-all">Refresh</button>
            <button class="btn tiny" @click="openFullChartFromCurrentNewTab" data-testid="btn-open-full-chart">Full chart</button>
          </div>
        </div>
      </div>

      <!-- ===== CHART ===== -->
      <section class="section chart-section" aria-label="Chart section">
        <!-- Chart toolbar -->
        <div class="chart-controls" role="toolbar" aria-label="Chart navigation">
          <div class="toolbar">
            <button class="btn micro" @click="panLeft" data-testid="btn-pan-left" aria-label="Pan left">←</button>
            <button class="btn micro" @click="zoomOut" data-testid="btn-zoom-out" aria-label="Zoom out">−</button>
            <button class="btn micro" @click="resetView" data-testid="btn-reset-view" aria-label="Reset view">Reset</button>
            <button class="btn micro" @click="zoomIn" data-testid="btn-zoom-in" aria-label="Zoom in">+</button>
            <button class="btn micro" @click="panRight" data-testid="btn-pan-right" aria-label="Pan right">→</button>
          </div>

          <div class="spacer"></div>

          <label for="tf-select">TF</label>
          <select v-model="ui.tf" class="cc-select" id="tf-select" data-testid="select-tf" aria-label="Timeframe">
            <option>1m</option><option>5m</option><option>15m</option>
            <option>1h</option><option>4h</option><option>1d</option>
          </select>

          <label class="cc-check"><input type="checkbox" v-model="ui.logScale" /> Log</label>
        </div>

        <!-- Chart block with overlay -->
        <div class="chart-block" :aria-busy="loading.price ? 'true' : 'false'" data-testid="chart-block">
          <div class="chart-overlay">
            <button class="btn micro ghost" title="Refresh chart" @click="refreshChart" data-testid="btn-refresh-chart">Refresh</button>
            <button class="btn micro ghost" title="Open full chart" @click="openFullChartFromCurrentNewTab">Full chart</button>
          </div>

          <WidgetShell @retry="reloadChart++" class="chart-shell">
            <!-- Chart is now rendered by ChartCard (emitter), NOT MarketChart -->
            <ChartCard
              :key="chartKey"
              :symbol="resolveApiTarget(activeClass, activeSymbol).symbol"
              :asset="resolveApiTarget(activeClass, activeSymbol).asset"
              :timeframe="ui.tf"
              :theme="theme"
              :height="'100%'"
              :candles="visibleCandles"
              :loading="loading.price"
              :error="errors.price"
              :chartType="marketChartType"
              :maxPoints="1200"
              :logScale="ui.logScale"
              :barSpacing="ui.barSpacing"
              :showVolume="ui.show.volume"
              @open="openFullChartFromCurrentNewTab"
              @crosshair="onCrosshair"
              @hover="onCrosshair"
              @providers="ingestProviders"
              @attribution="addProvidersFromText"
            />
          </WidgetShell>

          <!-- Accessible inline error, if any -->
          <div v-if="errors.price" class="chart-error" role="alert" aria-live="assertive" data-testid="chart-error">
            {{ errors.price }}
          </div>
        </div>

        <!-- OHLC panel -->
        <div class="ohlc-card" v-if="ohlc.ready" aria-live="polite" data-testid="ohlc-card">
          <div class="ohlc-row">
            <div class="ohlc-item"><span>Open</span><strong>{{ formatPrice(ohlc.open) }}</strong></div>
            <div class="ohlc-item"><span>High</span><strong>{{ formatPrice(ohlc.high) }}</strong></div>
            <div class="ohlc-item"><span>Low</span><strong>{{ formatPrice(ohlc.low) }}</strong></div>
            <div class="ohlc-item">
              <span>Close</span>
              <strong :class="ohlc.changePct>=0 ? 'up':'down'">
                {{ formatPrice(ohlc.close) }} ({{ signedPct(ohlc.changePct) }})
              </strong>
            </div>
            <div class="ohlc-item" v-if="Number.isFinite(ohlc.volume)"><span>Volume</span><strong>{{ formatVol(ohlc.volume) }}</strong></div>
            <div class="ohlc-item" v-if="ohlc.time"><span>Time</span><strong>{{ prettyTime(ohlc.time) }}</strong></div>
          </div>
        </div>
      </section>
    </main>

    <!-- ========== RIGHT RAIL ========== -->
    <aside class="right-rail" @click.stop aria-label="Secondary">
      <!-- Watchlist (moved ABOVE news) -->
      <div class="watch-card">
        <div class="watch-head">
          <h3 class="section-title small">Watchlist</h3>
          <div class="watch-actions">
            <button class="btn tiny" v-if="watchlist.length" @click="clearWatchlist" aria-label="Clear watchlist" data-testid="btn-clear-watch">Clear</button>
          </div>
        </div>

        <div v-if="!watchlist.length" class="empty small">No pins yet. Use Markets / Pairs / Top 4.</div>

        <ul v-else class="watch compact" data-testid="watchlist">
          <li
            v-for="w in pagedWatch"
            :key="w.key"
            class="watch-row compact"
            :class="{ active: isSelectedWatch(w) }"
            @click="quickSelect(w)"
            @dblclick="openFullChartFromWatchNewTab(w)"
            :title="${w.label} · ${formatPrice(latestPriceForWatch(w))} · ${signedPct(latestChangeForWatch(w))}"
          >
            <img class="row-ic" :src="iconFor(w.type, w.meta?.symbol || w.meta?.id || w.label)" :alt="w.label" @error="onIconError($event, w.type)" />
            <span class="row-name">{{ w.label }}</span>
            <span class="row-price">{{ formatPrice(latestPriceForWatch(w)) }}</span>
            <span class="row-chg" :class="(latestChangeForWatch(w) ?? 0) >= 0 ? 'up' : 'down'">
              {{ signedPct(latestChangeForWatch(w)) }}
            </span>
            <button
              class="icon-btn pin"
              :class="{ active: inWatchlist(w.meta || w, w.type) }"
              @click.stop="toggleWatchlist(w.meta || w, w.type)"
              :title="inWatchlist(w.meta || w, w.type) ? 'Unpin' : 'Pin'"
              aria-label="Pin"
              :aria-pressed="inWatchlist(w.meta || w, w.type) ? 'true' : 'false'"
            >
              <svg viewBox="0 0 24 24" class="ic"><path d="M14 2l1 5 3 3-5 1-6 8-1-1 8-6 1-5z"/></svg>
            </button>
            <button class="wl-x" @click.stop="removeFromWatchlist(w)" title="Remove" aria-label="Remove from watchlist">✕</button>
          </li>
        </ul>

        <div class="pager compact" v-if="watchlistTotalPages > 1">
          <button class="pg-btn" :disabled="watchlistPage === 1" @click="watchlistPage--" aria-label="Previous page">‹</button>
          <span class="pg-ind">{{ watchlistPage }} / {{ watchlistTotalPages }}</span>
          <button class="pg-btn" :disabled="watchlistPage === watchlistTotalPages" @click="watchlistPage++" aria-label="Next page">›</button>
        </div>
      </div>

      <!-- Live/Global News with COUNT TABS + RELOAD -->
      <div class="news-card rail">
        <div class="rail-head">
          <h3 class="section-title small">News</h3>

          <!-- PATCH A: Count tabs for both news feeds -->
          <div class="tabs small count-tabs">
            <button
              :class="{ active: newsTab==='relevant' }"
              @click="newsTab='relevant'"
              data-testid="tab-news-relevant"
            >
              Relevant
              <span class="badge">{{ relevantCount }}</span>
            </button>
            <button
              :class="{ active: newsTab==='global' }"
              @click="newsTab='global'"
              data-testid="tab-news-global"
            >
              Global
              <span class="badge">{{ globalCount }}</span>
            </button>
          </div>

          <!-- PATCH B: One-click reload for BOTH lists -->
          <div class="watch-actions">
            <button class="btn tiny" @click="reloadNews" title="Reload both news lists" data-testid="btn-reload-news">Reload</button>
          </div>
        </div>

        <!-- ===== Relevant (selection-specific) ===== -->
        <template v-if="newsTab==='relevant'">
          <div class="rail-subtitle">Live — {{ selectedLabel }}</div>

          <div v-if="newsError" class="news-error" role="alert" aria-live="assertive" data-testid="news-error">{{ newsError }}</div>

          <!-- Skeleton while loading -->
          <ul v-if="newsLoading" class="news-list rail">
            <li class="news-item rail" v-for="i in 3" :key="'nsk'+i">
              <div class="skeleton-row"></div>
              <div class="meta-row">
                <span class="chip ghost skeleton-chip"></span>
                <span class="chip ghost skeleton-chip"></span>
              </div>
            </li>
          </ul>

          <div v-else class="news-live" aria-live="polite">
            <template v-if="!newsError">
              <ul v-if="filteredNewsItems.length" class="news-list rail" data-testid="news-list">
                <li v-for="n in visibleNews" :key="n.id" class="news-item rail">
                  <a class="news-link clamp-2" :href="n.url" target="_blank" rel="noopener">{{ n.headline }}</a>
                  <div class="meta-row">
                    <span class="chip">{{ n.source }}</span>
                    <span class="dot">·</span>
                    <span class="chip ghost">{{ prettyTime(n.datetime) }}</span>
                  </div>
                </li>
              </ul>
              <div v-else-if="newsHintLink" class="news-hint" data-testid="news-hint">
                No direct headlines. Try <a :href="newsHintLink" target="_blank" rel="noopener">Google News</a>.
              </div>
              <div v-else class="news-empty">No news available.</div>

              <div v-if="filteredNewsItems.length > newsLimit" class="news-more">
                <button class="btn tiny" @click="newsLimit += 10">Show more</button>
              </div>
            </template>
          </div>

          <div v-if="newsAttribution" class="news-attr">{{ newsAttribution }}</div>
        </template>

        <!-- ===== Global (asset-wide) ===== -->
        <template v-else>
          <div class="rail-subtitle">Global — {{ prettyMarket(normalizeAssetParam(activeClass)) }}</div>

          <div v-if="globalNewsError" class="news-error" role="alert" aria-live="assertive" data-testid="global-news-error">{{ globalNewsError }}</div>

          <!-- Skeleton while loading -->
          <ul v-if="globalNewsLoading" class="news-list rail">
            <li class="news-item rail" v-for="i in 3" :key="'gnsk'+i">
              <div class="skeleton-row"></div>
              <div class="meta-row">
                <span class="chip ghost skeleton-chip"></span>
                <span class="chip ghost skeleton-chip"></span>
              </div>
            </li>
          </ul>

          <div v-else class="news-live" aria-live="polite">
            <template v-if="!globalNewsError">
              <ul v-if="visibleGlobalNews.length" class="news-list rail" data-testid="news-list-global">
                <li v-for="n in visibleGlobalNews" :key="n.id" class="news-item rail">
                  <a class="news-link clamp-2" :href="n.url" target="_blank" rel="noopener">{{ n.headline }}</a>
                  <div class="meta-row">
                    <span class="chip">{{ n.source }}</span>
                    <span class="dot">·</span>
                    <span class="chip ghost">{{ prettyTime(n.datetime) }}</span>
                  </div>
                </li>
              </ul>
              <div v-else-if="globalNewsHintLink" class="news-hint" data-testid="global-news-hint">
                No global headlines. Try <a :href="globalNewsHintLink" target="_blank" rel="noopener">Google News</a>.
              </div>
              <div v-else class="news-empty">No news available.</div>

              <div v-if="globalNewsItems.length > globalNewsLimit" class="news-more">
                <button class="btn tiny" @click="globalNewsLimit += 10">Show more</button>
              </div>
            </template>
          </div>

          <div v-if="globalNewsAttribution" class="news-attr">{{ globalNewsAttribution }}</div>
        </template>
      </div>

      <!-- Right Overlay (PAIRS only) -->
      <transition name="drop">
        <div
          v-if="rightOverlay.open"
          class="right-overlay"
          :class="{ floating: isCompact }"
          role="dialog"
          aria-modal="true"
          :aria-labelledby="rightOverlay.kind ? (rightOverlay.kind + '-title') : undefined"
          @keydown.esc="closeOverlay"
          tabindex="0"
          ref="overlayEl"
          data-testid="right-overlay"
        >
          <div class="ro-head">
            <div class="ro-title" :id="rightOverlay.kind ? (rightOverlay.kind + '-title') : null">
              <template v-if="rightOverlay.kind==='pairs'">Pair &amp; Selectors</template>
            </div>
            <div class="ro-actions">
              <button class="btn tiny" @click="closeOverlay">Close</button>
            </div>
          </div>

          <div v-if="rightOverlay.kind==='pairs'" class="ro-body">
            <div class="tabs small" style="margin-bottom:8px;">
              <button :class="{ active: activeTab === 'forex' }" @click="setTab('forex')" data-testid="tab-forex">Forex</button>
              <button :class="{ active: activeTab === 'crypto' }" @click="setTab('crypto')" data-testid="tab-crypto">Crypto</button>
            </div>

            <div class="list">
              <template v-if="activeTab === 'forex'">
                <div
                  v-for="p in fxPairs"
                  :key="p.symbol"
                  class="row"
                  @click="selectPair(p, 'fx')"
                  @dblclick="openFullChartNewTab({ key: p.symbol, label: p.symbol, class: 'fx' })"
                >
                  <img class="row-ic" :src="p.icon" :alt="p.symbol" @error="onIconError($event, 'fx')" />
                  <span class="row-name">{{ p.symbol }}</span>
                  <button
                    class="icon-btn pin"
                    :class="{ active: inWatchlist(p, 'fx') }"
                    @click.stop="toggleWatchlist(p, 'fx')"
                    :title="inWatchlist(p, 'fx') ? 'Unpin' : 'Pin'"
                    aria-label="Pin"
                    :aria-pressed="inWatchlist(p, 'fx') ? 'true' : 'false'"
                  >
                    <svg viewBox="0 0 24 24" class="ic"><path d="M14 2l1 5 3 3-5 1-6 8-1-1 8-6 1-5z"/></svg>
                  </button>
                </div>
              </template>

              <template v-else>
                <div
                  v-for="c in cryptoCoins"
                  :key="c.id"
                  class="row"
                  @click="selectPair({ id: c.id, symbol: c.symbol, name: c.name }, 'crypto')"
                  @dblclick="openFullChartNewTab({ key: c.id, label: c.symbol, class: 'crypto' })"
                >
                  <img class="row-ic" :src="c.icon" :alt="c.symbol" @error="onIconError($event, 'crypto')" />
                  <span class="row-name">{{ c.symbol }}</span>
                  <button
                    class="icon-btn pin"
                    :class="{ active: inWatchlist(c, 'crypto') }"
                    @click.stop="toggleWatchlist(c, 'crypto')"
                    :title="inWatchlist(c, 'crypto') ? 'Unpin' : 'Pin'"
                    aria-label="Pin"
                    :aria-pressed="inWatchlist(c, 'crypto') ? 'true' : 'false'"
                  >
                    <svg viewBox="0 0 24 24" class="ic"><path d="M14 2l1 5 3 3-5 1-6 8-1-1 8-6 1-5z"/></svg>
                  </button>
                </div>
              </template>
            </div>
          </div>
        </div>
      </transition>
    </aside>

    <!-- Mount Zeeno once; it manages its own floating window -->
    <KeepAlive>
      <Zeeno v-bind="zeenoProps" />
    </KeepAlive>
  </div>
</template>
      <!-- KPI + AI Overview + Actions -->
      <div class="kpi-row">
        <div class="kpi-card">
          <div class="kpi-title">Selected</div>
          <div class="kpi-selected">
            <div class="kpi-value">
              <img
                class="title-ic"
                :src="iconFor(activeClass, activeClass==='crypto' ? (activeName || activeSymbol) : activeSymbol)"
                :alt="selectedLabel"
                @error="onIconError($event, activeClass)"
              />
              <span>{{ selectedLabel }}</span>
            </div>
            <div class="kpi-sub" :class="(changeForSelected ?? 0) >= 0 ? 'up' : 'down'">
              {{ signedPct(changeForSelected) }}
            </div>
          </div>
        </div>

        <div class="kpi-card highlight">
          <div class="kpi-title">AI Overview — {{ selectedLabel }}</div>
          <div class="ai-overview" v-if="aiOverview.ready">
            <div class="ao-headline">{{ aiOverview.title }}</div>
            <div class="ao-tags">
              <span class="tag" :class="aiOverview.momentumClass">{{ aiOverview.momentum }}</span>
              <span class="tag" :class="aiOverview.regimeClass">{{ aiOverview.regime }}</span>
              <span class="tag">{{ aiOverview.volatility }}</span>
              <span class="tag ghost">live</span>
            </div>
            <p class="ao-summary">{{ aiOverview.summary }}</p>
            <ul class="ao-bullets">
              <li v-for="(b, i) in aiOverview.bullets" :key="i">{{ b }}</li>
            </ul>
            <div class="ao-foot"><span class="muted">Zeeno read:</span> {{ aiOverview.zeenoTake }}</div>
          </div>
          <div class="ai-overview empty" v-else>
            <div class="ao-headline">Gathering data…</div>
            <p class="ao-summary">Waiting for enough candles to compile a session recap and live trend read.</p>
          </div>
        </div>

        <div class="kpi-card">
          <div class="kpi-title">Actions</div>
          <div class="kpi-actions spaced" role="group" aria-label="Chart actions">
            <button class="btn tiny" @click="refreshAll" data-testid="btn-refresh-all">Refresh</button>
            <button class="btn tiny" @click="openFullChartFromCurrentNewTab" data-testid="btn-open-full-chart">Full chart</button>
          </div>
        </div>
      </div>

      <!-- ===== CHART ===== -->
      <section class="section chart-section" aria-label="Chart section">
        <!-- Chart toolbar -->
        <div class="chart-controls" role="toolbar" aria-label="Chart navigation">
          <div class="toolbar">
            <button class="btn micro" @click="panLeft" data-testid="btn-pan-left" aria-label="Pan left">←</button>
            <button class="btn micro" @click="zoomOut" data-testid="btn-zoom-out" aria-label="Zoom out">−</button>
            <button class="btn micro" @click="resetView" data-testid="btn-reset-view" aria-label="Reset view">Reset</button>
            <button class="btn micro" @click="zoomIn" data-testid="btn-zoom-in" aria-label="Zoom in">+</button>
            <button class="btn micro" @click="panRight" data-testid="btn-pan-right" aria-label="Pan right">→</button>
          </div>

          <div class="spacer"></div>

          <label for="tf-select">TF</label>
          <select v-model="ui.tf" class="cc-select" id="tf-select" data-testid="select-tf" aria-label="Timeframe">
            <option>1m</option><option>5m</option><option>15m</option>
            <option>1h</option><option>4h</option><option>1d</option>
          </select>

          <label class="cc-check"><input type="checkbox" v-model="ui.logScale" /> Log</label>
        </div>

        <!-- Chart block with overlay -->
        <div class="chart-block" :aria-busy="loading.price ? 'true' : 'false'" data-testid="chart-block">
          <div class="chart-overlay">
            <button class="btn micro ghost" title="Refresh chart" @click="refreshChart" data-testid="btn-refresh-chart">Refresh</button>
            <button class="btn micro ghost" title="Open full chart" @click="openFullChartFromCurrentNewTab">Full chart</button>
          </div>

          <WidgetShell @retry="reloadChart++" class="chart-shell">
            <ChartCard
              :key="chartKey"
              :symbol="resolveApiTarget(activeClass, activeSymbol).symbol"
              :asset="resolveApiTarget(activeClass, activeSymbol).asset"
              :timeframe="ui.tf"
              :theme="theme"
              :height="'100%'"
              :candles="visibleCandles"
              :loading="loading.price"
              :error="errors.price"
              :chartType="marketChartType"
              :maxPoints="1200"
              :logScale="ui.logScale"
              :barSpacing="ui.barSpacing"
              :showVolume="ui.show.volume"
              @open="openFullChartFromCurrentNewTab"
              @crosshair="onCrosshair"
              @hover="onCrosshair"
              @providers="ingestProviders"
              @attribution="addProvidersFromText"
            />
          </WidgetShell>

          <!-- Accessible inline error, if any -->
          <div v-if="errors.price" class="chart-error" role="alert" aria-live="assertive" data-testid="chart-error">
            {{ errors.price }}
          </div>
        </div>

        <!-- OHLC panel -->
        <div class="ohlc-card" v-if="ohlc.ready" aria-live="polite" data-testid="ohlc-card">
          <div class="ohlc-row">
            <div class="ohlc-item"><span>Open</span><strong>{{ formatPrice(ohlc.open) }}</strong></div>
            <div class="ohlc-item"><span>High</span><strong>{{ formatPrice(ohlc.high) }}</strong></div>
            <div class="ohlc-item"><span>Low</span><strong>{{ formatPrice(ohlc.low) }}</strong></div>
            <div class="ohlc-item">
              <span>Close</span>
              <strong :class="ohlc.changePct>=0 ? 'up':'down'">
                {{ formatPrice(ohlc.close) }} ({{ signedPct(ohlc.changePct) }})
              </strong>
            </div>
            <div class="ohlc-item" v-if="Number.isFinite(ohlc.volume)"><span>Volume</span><strong>{{ formatVol(ohlc.volume) }}</strong></div>
            <div class="ohlc-item" v-if="ohlc.time"><span>Time</span><strong>{{ prettyTime(ohlc.time) }}</strong></div>
          </div>
        </div>
      </section>
    </main>

    <!-- ========== RIGHT RAIL ========== -->
    <aside class="right-rail" @click.stop aria-label="Secondary">
      <!-- Watchlist (moved ABOVE news) -->
      <div class="watch-card">
        <div class="watch-head">
          <h3 class="section-title small">Watchlist</h3>
          <div class="watch-actions">
            <button class="btn tiny" v-if="watchlist.length" @click="clearWatchlist" aria-label="Clear watchlist" data-testid="btn-clear-watch">Clear</button>
          </div>
        </div>

        <div v-if="!watchlist.length" class="empty small">No pins yet. Use Markets / Pairs / Top 4.</div>

        <ul v-else class="watch compact" data-testid="watchlist">
          <li
            v-for="w in pagedWatch"
            :key="w.key"
            class="watch-row compact"
            :class="{ active: isSelectedWatch(w) }"
            @click="quickSelect(w)"
            @dblclick="openFullChartFromWatchNewTab(w)"
            :title="${w.label} · ${formatPrice(latestPriceForWatch(w))} · ${signedPct(latestChangeForWatch(w))}"
          >
            <img class="row-ic" :src="iconFor(w.type, w.meta?.symbol || w.meta?.id || w.label)" :alt="w.label" @error="onIconError($event, w.type)" />
            <span class="row-name">{{ w.label }}</span>
            <span class="row-price">{{ formatPrice(latestPriceForWatch(w)) }}</span>
            <span class="row-chg" :class="(latestChangeForWatch(w) ?? 0) >= 0 ? 'up' : 'down'">
              {{ signedPct(latestChangeForWatch(w)) }}
            </span>
            <button
              class="icon-btn pin"
              :class="{ active: inWatchlist(w.meta || w, w.type) }"
              @click.stop="toggleWatchlist(w.meta || w, w.type)"
              :title="inWatchlist(w.meta || w, w.type) ? 'Unpin' : 'Pin'"
              aria-label="Pin"
              :aria-pressed="inWatchlist(w.meta || w, w.type) ? 'true' : 'false'"
            >
              <svg viewBox="0 0 24 24" class="ic"><path d="M14 2l1 5 3 3-5 1-6 8-1-1 8-6 1-5z"/></svg>
            </button>
            <button class="wl-x" @click.stop="removeFromWatchlist(w)" title="Remove" aria-label="Remove from watchlist">✕</button>
          </li>
        </ul>

        <div class="pager compact" v-if="watchlistTotalPages > 1">
          <button class="pg-btn" :disabled="watchlistPage === 1" @click="watchlistPage--" aria-label="Previous page">‹</button>
          <span class="pg-ind">{{ watchlistPage }} / {{ watchlistTotalPages }}</span>
          <button class="pg-btn" :disabled="watchlistPage === watchlistTotalPages" @click="watchlistPage++" aria-label="Next page">›</button>
        </div>
      </div>

      <!-- News (tabs with counts + reload for both) -->
      <div class="news-card rail">
        <div class="rail-head" style="gap:8px;">
          <h3 class="section-title small">News</h3>
          <div class="tabs small" style="margin-left:auto;">
            <button :class="{ active: newsTab==='relevant' }" @click="newsTab='relevant'" data-testid="tab-news-relevant">
              Relevant <span class="chip ghost">{{ filteredNewsItems.length }}</span>
            </button>
            <button :class="{ active: newsTab==='global' }" @click="newsTab='global'; if(!globalNewsItems.length) refreshGlobalNews()" data-testid="tab-news-global">
              Global <span class="chip ghost">{{ globalNewsItems.length }}</span>
            </button>
            <button class="btn tiny" @click="reloadNewsBoth" data-testid="btn-reload-news">Reload</button>
          </div>
        </div>

        <!-- Tab: Relevant (by selection) -->
        <template v-if="newsTab==='relevant'">
          <div v-if="newsError" class="news-error" role="alert" aria-live="assertive" data-testid="news-error">{{ newsError }}</div>

          <!-- Skeleton while loading -->
          <ul v-if="newsLoading" class="news-list rail">
            <li class="news-item rail" v-for="i in 3" :key="'nsk'+i">
              <div class="skeleton-row"></div>
              <div class="meta-row">
                <span class="chip ghost skeleton-chip"></span>
                <span class="chip ghost skeleton-chip"></span>
              </div>
            </li>
          </ul>

          <div v-else class="news-live" aria-live="polite">
            <template v-if="!newsError">
              <ul v-if="filteredNewsItems.length" class="news-list rail" data-testid="news-list">
                <li v-for="n in visibleNews" :key="n.id" class="news-item rail">
                  <a class="news-link clamp-2" :href="n.url" target="_blank" rel="noopener">{{ n.headline }}</a>
                  <div class="meta-row">
                    <span class="chip">{{ n.source }}</span>
                    <span class="dot">·</span>
                    <span class="chip ghost">{{ prettyTime(n.datetime) }}</span>
                  </div>
                </li>
              </ul>
              <div v-else-if="newsHintLink" class="news-hint" data-testid="news-hint">
                No direct headlines. Try <a :href="newsHintLink" target="_blank" rel="noopener">Google News</a>.
              </div>
              <div v-else class="news-empty">No news available.</div>

              <div v-if="filteredNewsItems.length > newsLimit" class="news-more">
                <button class="btn tiny" @click="newsLimit += 10">Show more</button>
              </div>
            </template>
          </div>

          <div v-if="newsAttribution" class="news-attr">{{ newsAttribution }}</div>
        </template>

        <!-- Tab: Global (by asset class) -->
        <template v-else>
          <div v-if="globalNewsError" class="news-error" role="alert" aria-live="assertive" data-testid="global-news-error">{{ globalNewsError }}</div>

          <!-- Skeleton while loading -->
          <ul v-if="globalNewsLoading" class="news-list rail">
            <li class="news-item rail" v-for="i in 3" :key="'gnsk'+i">
              <div class="skeleton-row"></div>
              <div class="meta-row">
                <span class="chip ghost skeleton-chip"></span>
                <span class="chip ghost skeleton-chip"></span>
              </div>
            </li>
          </ul>

          <div v-else class="news-live" aria-live="polite">
            <template v-if="!globalNewsError">
              <ul v-if="visibleGlobalNews.length" class="news-list rail" data-testid="global-news-list">
                <li v-for="n in visibleGlobalNews" :key="n.id" class="news-item rail">
                  <a class="news-link clamp-2" :href="n.url" target="_blank" rel="noopener">{{ n.headline }}</a>
                  <div class="meta-row">
                    <span class="chip">{{ n.source }}</span>
                    <span class="dot">·</span>
                    <span class="chip ghost">{{ prettyTime(n.datetime) }}</span>
                  </div>
                </li>
              </ul>
              <div v-else class="news-empty">No global headlines available.</div>

              <div v-if="globalNewsItems.length > globalNewsLimit" class="news-more">
                <button class="btn tiny" @click="globalNewsLimit += 10">Show more</button>
              </div>
            </template>
          </div>

          <div v-if="globalNewsAttribution" class="news-attr">{{ globalNewsAttribution }}</div>
        </template>
      </div>

      <!-- Right Overlay (PAIRS only) -->
      <transition name="drop">
        <div
          v-if="rightOverlay.open"
          class="right-overlay"
          :class="{ floating: isCompact }"
          role="dialog"
          aria-modal="true"
          :aria-labelledby="rightOverlay.kind ? (rightOverlay.kind + '-title') : undefined"
          @keydown.esc="closeOverlay"
          tabindex="0"
          ref="overlayEl"
          data-testid="right-overlay"
        >
          <div class="ro-head">
            <div class="ro-title" :id="rightOverlay.kind ? (rightOverlay.kind + '-title') : null">
              <template v-if="rightOverlay.kind==='pairs'">Pair &amp; Selectors</template>
            </div>
            <div class="ro-actions">
              <button class="btn tiny" @click="closeOverlay">Close</button>
            </div>
          </div>

          <div v-if="rightOverlay.kind==='pairs'" class="ro-body">
            <div class="tabs small" style="margin-bottom:8px;">
              <button :class="{ active: activeTab === 'forex' }" @click="setTab('forex')" data-testid="tab-forex">Forex</button>
              <button :class="{ active: activeTab === 'crypto' }" @click="setTab('crypto')" data-testid="tab-crypto">Crypto</button>
            </div>

            <div class="list">
              <template v-if="activeTab === 'forex'">
                <div
                  v-for="p in fxPairs"
                  :key="p.symbol"
                  class="row"
                  @click="selectPair(p, 'fx')"
                  @dblclick="openFullChartNewTab({ key: p.symbol, label: p.symbol, class: 'fx' })"
                >
                  <img class="row-ic" :src="p.icon" :alt="p.symbol" @error="onIconError($event, 'fx')" />
                  <span class="row-name">{{ p.symbol }}</span>
                  <button
                    class="icon-btn pin"
                    :class="{ active: inWatchlist(p, 'fx') }"
                    @click.stop="toggleWatchlist(p, 'fx')"
                    :title="inWatchlist(p, 'fx') ? 'Unpin' : 'Pin'"
                    aria-label="Pin"
                    :aria-pressed="inWatchlist(p, 'fx') ? 'true' : 'false'"
                  >
                    <svg viewBox="0 0 24 24" class="ic"><path d="M14 2l1 5 3 3-5 1-6 8-1-1 8-6 1-5z"/></svg>
                  </button>
                </div>
              </template>

              <template v-else>
                <div
                  v-for="c in cryptoCoins"
                  :key="c.id"
                  class="row"
                  @click="selectPair({ id: c.id, symbol: c.symbol, name: c.name }, 'crypto')"
                  @dblclick="openFullChartNewTab({ key: c.id, label: c.symbol, class: 'crypto' })"
                >
                  <img class="row-ic" :src="c.icon" :alt="c.symbol" @error="onIconError($event, 'crypto')" />
                  <span class="row-name">{{ c.symbol }}</span>
                  <button
                    class="icon-btn pin"
                    :class="{ active: inWatchlist(c, 'crypto') }"
                    @click.stop="toggleWatchlist(c, 'crypto')"
                    :title="inWatchlist(c, 'crypto') ? 'Unpin' : 'Pin'"
                    aria-label="Pin"
                    :aria-pressed="inWatchlist(c, 'crypto') ? 'true' : 'false'"
                  >
                    <svg viewBox="0 0 24 24" class="ic"><path d="M14 2l1 5 3 3-5 1-6 8-1-1 8-6 1-5z"/></svg>
                  </button>
                </div>
              </template>
            </div>
          </div>
        </div>
      </transition>
    </aside>

    <!-- Mount Zeeno once; it manages its own floating window -->
    <KeepAlive>
      <Zeeno v-bind="zeenoProps" />
    </KeepAlive>
  </div>
</template>
<script>
/* eslint-disable no-console */
import { ref, reactive, computed, onMounted, onBeforeUnmount, watch, nextTick, defineAsyncComponent, getCurrentInstance } from 'vue'
import { useRouter } from 'vue-router'
import { signOut } from 'firebase/auth'
import { auth, authReady } from '@/firebase'
import { completeLogout } from '@/router'
import WidgetShell from '@/components/WidgetShell/WidgetShell.vue'
import ChartCard from '@/components/ChartCard/ChartCard.vue' // ⬅️ use ChartCard (emitter)
const Settings = defineAsyncComponent(() => import('@/views/Settings.vue'))
import Zeeno from '@/components/Zeeno/Zeeno.vue'

const tvEnabled = String(import.meta.env.VITE_TV_WIDGETS || '0') === '1'
const probeProvidersEnabled = String(import.meta.env.VITE_PROBE_PROVIDERS || '0') === '1'

import wickwiseEmblem from '@/assets/wickwise-emblem.svg'
import { fetchWithFallback } from '@/services/api.js'

const MAX_CANDLES_POP = Number(import.meta.env.VITE_ZEENO_POP_MAX_CANDLES || 400)

const hscroll = {
  mounted(el) {
    const onWheel = (e) => {
      if (Math.abs(e.deltaY) > Math.abs(e.deltaX)) {
        el.scrollLeft += e.deltaY
        e.preventDefault()
      }
    }
    el.addEventListener('wheel', onWheel, { passive: false })
    el._hscroll = onWheel
  },
  unmounted(el) {
    el.removeEventListener('wheel', el._hscroll)
  }
}

export default {
  name: 'Dashboard',
  components: { WidgetShell, ChartCard, Settings, Zeeno },
  directives: { hscroll },

  errorCaptured(err) {
    try { console.error('[Dashboard errorCaptured]', err) } catch {}
    const fn = (this && (this._setFatal || (this.$data && this.$data._setFatal)))
    if (typeof fn === 'function') {
      fn(err?.message || String(err || 'Unknown error'))
    }
    return false
  },

  setup() {
    // Test-safe router & navigation helpers
    const IS_TEST = typeof import.meta !== 'undefined' && import.meta.env?.MODE === 'test'
    let router = null
    try { router = useRouter() } catch { router = null }

    function safePush(to) {
      try { if (router && typeof router.push === 'function') return router.push(to) } catch {}
      return Promise.resolve()
    }

    const inst = getCurrentInstance()
    const fatalErrorRef = ref('')
    const _setFatal = (msg) => { fatalErrorRef.value = msg }
    if (inst && inst.proxy) { inst.proxy._setFatal = _setFatal }

    // ENV / API
    const backendKey = import.meta.env.VITE_BACKEND_SECRET_KEY || import.meta.env.VITE_BACKEND_KEY || ''
    const apiBase = (import.meta.env.VITE_API_BASE || '/api').replace(/\/+$/, '')
    const marketApiKey = import.meta.env.VITE_MARKET_API_KEY || ''
    const autoRefreshMs = Math.max(60000, Number(import.meta.env.VITE_AUTO_REFRESH_MS || 300000))

    // Providers / attribution
    const serverLiveData = ref(false)
    const providerStatus = reactive(Object.create(null))
    const PROVIDER_META = {
      polygon: { name: 'Polygon.io', icon: '/providers/polygon.svg' },
      finnhub: { name: 'Finnhub', icon: '/providers/finnhub.svg' },
      twelvedata: { name: 'Twelve Data', icon: '/providers/twelvedata.svg' },
      coingecko: { name: 'CoinGecko', icon: '/providers/coingecko.svg' },
      newsapi: { name: 'NewsAPI', icon: '/providers/newsapi.svg' },
      worldbank: { name: 'World Bank', icon: '/providers/worldbank.svg' },
      oanda: { name: 'OANDA', icon: '/providers/oanda.svg' },
      alphavantage: { name: 'Alpha Vantage', icon: '/providers/alphavantage.svg' },
      yahoo: { name: 'Yahoo Finance', icon: '/providers/yahoo.svg' },
      binance: { name: 'Binance', icon: '/providers/binance.svg' },
      coinapi: { name: 'CoinAPI', icon: '/providers/coinapi.svg' },
      cryptocompare: { name: 'CryptoCompare', icon: '/providers/cryptocompare.svg' },
      'wickwise-mock': { name: 'WickWise Mock', icon: '/providers/mock.svg' },
      default: { name: 'Data Provider', icon: '/providers/default.svg' },
    }

    function addProvider(id, ok = true, title = '') {
      const key = String(id || '').toLowerCase().trim()
      if (!key) return
      const prev = providerStatus[key] || {}
      providerStatus[key] = { ok: !!ok, title: title || prev.title || '', last: Date.now() }
    }

    function addProvidersFromText(text) {
      const s = String(text || '').toLowerCase()
      const tests = [
        ['polygon', /polygon/],
        ['finnhub', /finnhub/],
        ['twelvedata', /(twelve[\s-]?data|12data|twelvedata)/],
        ['coingecko', /coingecko/],
        ['newsapi', /newsapi/],
        ['worldbank', /(world\s*bank|wbapi)/],
        ['oanda', /oanda/],
        ['alphavantage', /(alpha[\s-]?vantage|av api)/],
        ['yahoo', /yahoo/],
        ['binance', /binance/],
        ['coinapi', /coinapi/],
        ['cryptocompare', /cryptocompare/],
      ]
      let matched = false
      for (const [id, re] of tests) {
        if (re.test(s)) { addProvider(id, true, text); matched = true }
      }
      if (!matched && s) addProvider('default', true, text)
    }

    // NEW: ingest providers payloads from ChartCard (@providers)
    function ingestProviders(payload) {
      if (!payload) return
      if (Array.isArray(payload)) {
        payload.forEach((p) => {
          if (typeof p === 'string') {
            addProvider(p, true, p)
          } else if (p && (p.id || p.name)) {
            const id = String(p.id || p.name).toLowerCase()
            addProvider(id, p.ok !== false, p.title || p.name || id)
          }
        })
        return
      }
      if (typeof payload === 'string') {
        addProvidersFromText(payload)
        return
      }
      if (payload && typeof payload === 'object') {
        if (Array.isArray(payload.providers)) ingestProviders(payload.providers)
        if (Array.isArray(payload.sources)) ingestProviders(payload.sources)
        const txt = payload.attribution || payload.provider || payload.source || ''
        if (txt) addProvidersFromText(txt)
      }
    }

    async function probeProviders() {
      if (!probeProvidersEnabled) return
      const hasReal = Object.keys(providerStatus).some(k => k !== 'default' && k !== 'wickwise-mock')
      if (hasReal) return

      const candidates = [ '/providers', '/market/providers', '/status/providers', '/about/providers', '/devkey/whoami' ]
      const normId = (s='') => String(s).toLowerCase().trim()

      function ingest(obj) {
        if (!obj) return false
        let any = false
        const addList = (arr) => {
          (arr || []).forEach(x => {
            if (typeof x === 'string') {
              addProvider(normId(x), true, x); any = true
            } else if (x && (x.id || x.name)) {
              const id = normId(x.id || x.name)
              addProvider(id, x.ok !== false, x.title || x.name || id); any = true
            }
          })
        }
        if (Array.isArray(obj)) { addList(obj); return any }
        if (obj.providers) { addList(obj.providers); any = true }
        if (obj.sources)   { addList(obj.sources);   any = true }
        if (obj.keys || obj.config) {
          const bag = obj.keys || obj.config
          Object.keys(bag).forEach(k => {
            const ok = !!bag[k]
            const id = normId(k)
            if (ok) { addProvider(id, true, id); any = true }
          })
        }
        const txt = obj.attribution || obj.provider || obj.source || obj.sourcesText || ''
        if (txt) { addProvidersFromText(txt); any = true }

        if (any && providerStatus['default']) delete providerStatus['default']
        return any
      }

      for (const path of candidates) {
        try {
          const j = await authed(path, { params: { brief: '1' } })
          if (ingest(j)) break
        } catch {}
      }
    }

    const providerChips = computed(() => {
      const ids = Object.keys(providerStatus)
      const chips = ids.map(id => {
        const meta = PROVIDER_META[id] || PROVIDER_META.default
        return {
          id,
          name: meta.name,
          icon: meta.icon,
          ok: !!providerStatus[id]?.ok,
          title: providerStatus[id]?.title || meta.name
        }
      })
      return chips.sort((a,b) => (a.ok === b.ok ? a.name.localeCompare(b.name) : a.ok ? -1 : 1))
    })

    function onProviderIconError(e, p) {
      try {
        if (e?.target?.getAttribute('data-fb')) { e.target.style.display = 'none'; return }
        e.target.src = PROVIDER_META.default.icon
        e.target.setAttribute('data-fb', '1')
      } catch {}
    }

    const serverAuthError = ref('')
    function flagAuthIssue(payload){
      const s = String(payload?.error || payload?.detail || payload?.message || payload || '')
      if (s.includes('api_key') || s.includes('401') || s.includes('403')) {
        serverAuthError.value = 'Client not authorised. Ensure VITE_BACKEND_SECRET_KEY (client) matches BACKEND_SECRET_KEY (server).'
      } else if (s.includes('400')) {
        serverAuthError.value = 'Request rejected by server (400). Check query params (symbol, asset, email) and keys.'
      }
    }

    const authed = (path, options = {}) => {
      const headers = { ...(options.headers || {}) }
      if (backendKey) {
        headers['x-backend-key'] = backendKey
        headers['authorization'] = Bearer ${backendKey}
      }
      const opts = { ...options, headers }
      return fetchWithFallback(path, opts)
        .then((j)=>{ if (j && (j.error||j.detail)) flagAuthIssue(j); return j })
        .catch((e)=>{ flagAuthIssue(e); throw e })
    }

    // ===== THEME =====
    const theme = ref('light'); try { theme.value = localStorage.getItem('theme') || 'light' } catch {}
    const setTheme = (v) => { theme.value = v; try { localStorage.setItem('theme', v) } catch {}; try { document.documentElement.setAttribute('data-theme', v) } catch {}; try { window.dispatchEvent(new Event('themechange')) } catch {} }
    const toggleTheme = () => { setTheme(theme.value === 'dark' ? 'light' : 'dark') }
    const instantToggleTheme = () => { toggleTheme() }
    let disconnectThemeObserver = null
    function initThemeSync(){
      if (typeof window === 'undefined' || typeof document === 'undefined') return
      const apply = () => { try { const t = document.documentElement.getAttribute('data-theme'); if (t === 'light' || t === 'dark') theme.value = t } catch {} }
      apply()
      try {
        const mo = new MutationObserver(apply)
        mo.observe(document.documentElement, { attributes: true, attributeFilter: ['data-theme'] })
        disconnectThemeObserver = () => mo.disconnect()
      } catch { disconnectThemeObserver = null }
    }

    // ===== DEFENSIVE AUTH GUARD =====
    const isAuthenticated = ref(true)
    async function ensureAuth(){
      let ok = true
      try { await authReady; ok = !!(auth && auth.currentUser) } catch { ok = true }
      if (!ok) {
        try { const tok = localStorage.getItem('authToken'); ok = !!tok } catch {}
      }
      isAuthenticated.value = ok
      if (!ok) {
        const redirect = router?.currentRoute?.value?.fullPath || '/dashboard'
        await safePush({ name: 'Login', query: { redirect } }).catch(()=>{})
        await safePush('/login').catch(()=>{})
      }
    }

    // ===== Subscription tier =====
    const devPlan = ref('free')
    const devLevel = ref(0)
    function labelForPlan(p){
      const canon = String(p || '').toLowerCase()
      if (canon === 'owner') return 'Owner'
      if (canon === 'zeeno_plus') return 'Zeeno Plus'
      if (canon === 'zeeno') return 'Zeeno'
      if (canon === 'gold') return 'Gold'
      return 'Free'
    }
    const devPlanLabel = computed(() => labelForPlan(devPlan.value))

    function currentUserEmail(){
      try { const e = auth?.currentUser?.email; if (e) return String(e).trim() } catch {}
      try { const e = localStorage.getItem('userEmail') || localStorage.getItem('email'); if (e) return String(e).trim() } catch {}
      return ''
    }

    async function fetchDevTier(){
      try {
        const email = currentUserEmail()
        if (!email) return
        const j = await authed('/devkey/whoami', { params: { email } })
        if (j && j.ok) {
          devPlan.value = String(j.plan || 'free').toLowerCase()
          const lvl = Number(j.level); devLevel.value = Number.isFinite(lvl) ? lvl : 0
          if (j.keys) {
            Object.keys(j.keys).forEach(k => { if (j.keys[k]) addProvider(k, true, 'Configured') })
          }
        }
      } catch { devPlan.value = 'free'; devLevel.value = 0 }
    }

    // ===== Sidebar/Overlays/Settings =====
    const sidebarCollapsed = ref(false); try { sidebarCollapsed.value = JSON.parse(localStorage.getItem('sidebarCollapsed') || 'false') } catch {}
    const persistSidebar = () => { try { localStorage.setItem('sidebarCollapsed', JSON.stringify(sidebarCollapsed.value)) } catch {} }

    const leftTab = ref('markets')
    const rightOverlay = reactive({ open:false, kind:null })
    const overlayEl = ref(null)

    const marketPanel = reactive({ open:false })
    const marketEl = ref(null)
    const marketsView = ref('list')

    const isCompact = ref(typeof window !== 'undefined' ? window.innerWidth < 1400 : false)
    function onResize(){ isCompact.value = window.innerWidth < 1400 }

    function openOverlay(kind){
      leftTab.value = kind
      rightOverlay.open = true
      rightOverlay.kind = kind
      if (kind === 'pairs') ensurePairsLoaded()
      nextTick(()=> overlayEl.value?.focus())
    }
    function closeOverlay(){ rightOverlay.open = false; rightOverlay.kind = null }

    function openMarketsPanel(){
      leftTab.value = 'markets'
      rightOverlay.open = false
      rightOverlay.kind = null
      marketsView.value = tvEnabled ? 'overview' : 'list'
      marketPanel.open = true
      nextTick(()=> marketEl.value?.focus())
    }
    function closeMarketsPanel(){ marketPanel.open = false }
    function openTop4InPanel(){
      leftTab.value = 'markets'
      marketPanel.open = true
      marketsView.value = 'top4'
      nextTick(loadTop4ForMarket)
    }

    const settingsOpen = ref(false)
    const settingsEl = ref(null)
    function toggleSettings(){
      settingsOpen.value = !settingsOpen.value
      if (settingsOpen.value) {
        leftTab.value = 'settings'
        nextTick(()=> settingsEl.value?.focus())
      } else {
        if (leftTab.value === 'settings') leftTab.value = 'markets'
        try { window.scrollTo({ top: 0, behavior: 'smooth' }) } catch {}
      }
    }
    function onOpenSettingsBridge(){ settingsOpen.value = true; leftTab.value = 'settings'; nextTick(()=> settingsEl.value?.focus()) }

    // Chart type dropdown
    const chartMenuOpen = ref(false)
    function toggleChartTypeMenu(){ chartMenuOpen.value = !chartMenuOpen.value }

    const ui = reactive({
      chartMode: (typeof localStorage !== 'undefined' && localStorage.getItem('ui.chartMode')) || 'candlestick',
      tf: ((typeof localStorage !== 'undefined' && localStorage.getItem('ui.tf')) || '1h').toLowerCase(),
      logScale: JSON.parse((typeof localStorage !== 'undefined' && localStorage.getItem('ui.logScale')) || 'false'),
      show: { volume: JSON.parse((typeof localStorage !== 'undefined' && localStorage.getItem('ui.show.volume')) || 'true') },
      barSpacing: Number((typeof localStorage !== 'undefined' && localStorage.getItem('ui.barSpacing')) || 6)
    })

    function setChartMode(mode){ ui.chartMode = mode; try { localStorage.setItem('ui.chartMode', mode) } catch {}; chartMenuOpen.value = false }
    watch(() => ui.tf, v => { try { localStorage.setItem('ui.tf', v) } catch {} })
    watch(() => ui.logScale, v => { try { localStorage.setItem('ui.logScale', JSON.stringify(v)) } catch {} })
    watch(() => ui.show.volume, v => { try { localStorage.setItem('ui.show.volume', JSON.stringify(v)) } catch {} })
    watch(() => ui.barSpacing, v => { try { localStorage.setItem('ui.barSpacing', String(v)) } catch {} })

    // Named popouts
    let _zeenoPopGate = false
    function openNamedZeenoPopout(url, name = 'WW-Zeeno') {
      if (_zeenoPopGate) return
      _zeenoPopGate = true
      setTimeout(() => { _zeenoPopGate = false }, 700)
      let w = null
      try { w = window.open('', name) } catch {}
      const features = 'popup=1,width=1120,height=760,menubar=no,toolbar=no,location=no,status=no,scrollbars=yes,resizable=yes'
      if (w && !w.closed) { try { w.focus(); w.location.replace(url); return } catch {} }
      window.open(url, name, features)
    }

    let _chartPopGate = false
    function openNamedChartPopout(url, name = 'WW-Chart') {
      if (_chartPopGate) return
      _chartPopGate = true
      setTimeout(() => { _chartPopGate = false }, 700)
      let w = null
      try { w = window.open('', name) } catch {}
      const features = 'popup=1,width=1280,height=820,menubar=no,toolbar=no,location=no,status=no,scrollbars=yes,resizable=yes'
      if (w && !w.closed) { try { w.focus(); w.location.replace(url); return } catch {} }
      window.open(url, name, features + ',noopener')
    }

    function openRouteInNewTabByName(name, query = {}) {
      try {
        const { href } = router.resolve({ name, query })
        if (name === 'FullChart') {
          openNamedChartPopout(href, 'WW-Chart')
        } else if (name === 'ZeenoPopout') {
          openNamedZeenoPopout(href, 'WW-Zeeno')
        } else {
          const w = window.open(href, '_blank', 'noopener,noreferrer')
          if (!w) throw new Error('popup-blocked')
          try { w.opener = null } catch {}
        }
      } catch {
        const qs = new URLSearchParams(query).toString()
        if (name === 'FullChart') {
          openNamedChartPopout('/fullchart' + (qs ? ?${qs} : ''), 'WW-Chart')
        } else {
          const w = window.open((name === 'ZeenoPopout' ? '/zeeno' : '/') + (qs ? ?${qs} : ''), '_blank', 'noopener,noreferrer')
          if (w) try { w.opener = null } catch {}
        }
      }
    }

    function openHomeInNewTab(){
      try {
        const { href } = router.resolve({ path: '/' })
        const w = window.open(href, '_blank', 'noopener,noreferrer')
        if (w) try { w.opener = null } catch {}
      } catch {}
    }

    // === POP-OUT STABILITY PATCH: safe JSON + bounded candles ===
    function safeStringify(obj) { try { return JSON.stringify(obj) } catch { return '' } }
    function byteLen(str='') { try { return new Blob([str]).size } catch { return (str.length || 0) * 2 } }
    function safeStore(key, payload, opts = {}) {
      const maxBytes = Number(opts.maxBytes ?? 1_200_000)
      let p = payload
      const attempts = [
        (x) => x,
        (x) => ({ ...x, candles: Array.isArray(x.candles) ? x.candles.slice(-MAX_CANDLES_POP) : [] }),
        (x) => ({ ...x, candles: Array.isArray(x.candles) ? x.candles.slice(-120) : [] }),
        (x) => ({ ...x, candles: [] }),
      ]
      for (const fn of attempts) {
        const candidate = fn(p)
        const str = safeStringify(candidate)
        if (!str) continue
        if (byteLen(str) <= maxBytes) {
          try { localStorage.setItem(key, str); return true } catch {}
        }
      }
      try { localStorage.setItem(key, safeStringify({ ...payload, candles: [], note: 'trimmed:localStorage-quota' })); return true } catch {}
      return false
    }

    function buildZeenoPopoutPayload(){
      const trimmed = Array.isArray(currentCandles.value) ? currentCandles.value.slice(-MAX_CANDLES_POP) : []
      return {
        selectedPair: { symbol: activeSymbol.value, name: activeName.value || activeSymbol.value, type: activeClass.value },
        timeframe: ui.tf,
        candles: trimmed,
        lastCandle: lastCandle.value,
        subscriptionTier: devPlanLabel.value,
        apiBase,
        userId: userId.value,
        welcomeText: 'Hi! Ask me about this chart or say /full for a full-chart scan.',
        apiKey: marketApiKey,
        serverKey: backendKey || ''
      }
    }
    function persistZeenoPopoutPayload(sid, payload){
      if (!sid) return
      try {
        safeStore(zeeno:popout:${sid}, payload)
        safeStore(wickbot:popout:${sid}, payload)
      } catch {}
    }
    function openZeenoNewTab(){
      const sid = (Math.random().toString(36).slice(2, 8) + Date.now().toString(36)).toUpperCase()
      const payload = buildZeenoPopoutPayload()
      persistZeenoPopoutPayload(sid, payload)
      let href = ''
      try {
        href = router.resolve({
          name: 'ZeenoPopout',
          query: { symbol: activeSymbol.value, type: activeClass.value, timeframe: ui.tf, sid, autoAnalyze: '1', api: apiBase, user: userId.value, theme: theme.value }
        }).href
      } catch {
        const q = new URLSearchParams({
          symbol: activeSymbol.value, type: activeClass.value, timeframe: ui.tf, sid, autoAnalyze: '1', api: apiBase, user: userId.value, theme: theme.value
        }).toString()
        href = '/zeeno?' + q
      }
      openNamedZeenoPopout(href, 'WW-Zeeno')
      closeAllTransientUI()
    }

    // Selection (persist)
    const activeSymbol = ref('EURUSD')
    const activeName = ref('EURUSD')
    const activeClass = ref('fx')
    try {
      const ps = localStorage.getItem('sel.symbol')
      const pc = localStorage.getItem('sel.class')
      const pn = localStorage.getItem('sel.name')
      if (ps && pc) { activeSymbol.value = ps; activeClass.value = pc; activeName.value = pn || ps }
    } catch {}
    watch([activeSymbol, activeClass, activeName], () => {
      try {
        localStorage.setItem('sel.symbol', activeSymbol.value)
        localStorage.setItem('sel.class', activeClass.value)
        localStorage.setItem('sel.name', activeName.value || activeSymbol.value)
      } catch {}
    })
    const selectedLabel = computed(() => activeName.value || activeSymbol.value)

    // Cursor / OHLC
    const ohlc = reactive({ ready:false, open:NaN, high:NaN, low:NaN, close:NaN, volume:NaN, time:null, changePct:0 })
    const lastCandle = ref(null)

    const lastPrevClose = computed(() => {
      const cs = currentCandles.value || []
      const n = cs.length
      const last = cs[n - 1]
      const prev = cs[n - 2]
      return prev ? Number(prev.close) : (last ? Number(last.open) : NaN)
    })

    function updateCursorFromCandle(c){
      if (!c) return
      const prev = Number(lastPrevClose.value)
      const close = Number(c.close)
      const ch = (Number.isFinite(close) && Number.isFinite(prev) && prev !== 0) ? ((close - prev)/prev)*100 : 0
      ohlc.open = c.open; ohlc.high = c.high; ohlc.low = c.low; ohlc.close = c.close; ohlc.volume = c.volume ?? c.v; ohlc.time = c.time
      ohlc.changePct = ch; ohlc.ready = true
    }

    function indexClosestByTime(arr, t){
      if (!arr || !arr.length) return -1
      let lo = 0, hi = arr.length - 1, best = 0
      while (lo <= hi) {
        const mid = (lo + hi) >> 1
        const mt = arr[mid].time
        if (mt === t) { best = mid; break }
        if (mt < t) { best = mid; lo = mid + 1 } else { hi = mid - 1 }
      }
      const a = arr[best]
      const b = arr[Math.min(arr.length - 1, best + 1)]
      return Math.abs((a?.time||0) - t) <= Math.abs((b?.time||0) - t) ? best : Math.min(arr.length - 1, best + 1)
    }

    let raf = 0
    let pendingHover = null
    function applyHoverFromPayload(p){
      if (!p) return
      // Direct OHLC payload
      if (p.open != null && p.high != null && p.low != null && p.close != null) {
        const prev = Number(p.prevClose)
        const c = Number(p.close)
        const ch = (Number.isFinite(c) && Number.isFinite(prev) && prev !== 0) ? ((c - prev)/prev)*100 : 0
        ohlc.open = p.open; ohlc.high = p.high; ohlc.low = p.low; ohlc.close = p.close; ohlc.volume = p.volume; ohlc.time = p.time
        ohlc.changePct = ch; ohlc.ready = true
        return
      }
      // Crosshair time + price
      const t = Number(p.time)
      const cs = currentCandles.value || []
      if (!cs.length || !Number.isFinite(t)) return
      const idx = indexClosestByTime(cs, t)
      if (idx < 0) return
      const bar = cs[idx]
      const prev = cs[idx - 1]
      const close = Number.isFinite(p.priceAtCursor) ? Number(p.priceAtCursor) : Number(bar.close)
      const prevClose = Number(prev?.close)
      const ch = (Number.isFinite(close) && Number.isFinite(prevClose) && prevClose !== 0) ? ((close - prevClose)/prevClose)*100 : 0
      ohlc.open = bar.open; ohlc.high = bar.high; ohlc.low = bar.low; ohlc.close = close; ohlc.volume = bar.volume ?? bar.v; ohlc.time = bar.time
      ohlc.changePct = ch; ohlc.ready = true
    }
    function onCrosshair(payload){
      pendingHover = payload
      if (raf) return
      raf = requestAnimationFrame(() => { raf = 0; const p = pendingHover; pendingHover = null; applyHoverFromPayload(p) })
    }
    function onGlobalCrosshair(e){
      const d = e?.detail || {}
      thead: // (no-op label to keep linters happy)
      {
        const sym = String(d.symbol || '').toUpperCase()
        const cur = String(activeSymbol.value || '').toUpperCase()
        if (!sym || sym !== cur) break thead
        onCrosshair({ time: Number(d.time), priceAtCursor: Number(d.price) })
      }
    }

    watch([() => activeSymbol.value, () => currentCandles.value], () => {
      const lc = currentCandles.value[currentCandles.value.length - 1]
      if (lc) updateCursorFromCandle(lc)
    })

    // Watchlist
    const WATCH_PAGE_SIZE = 14
    const watchlist = ref(JSON.parse(localStorage.getItem('watchlist') || '[]'))
    const syncWatchlist = () => { try { localStorage.setItem('watchlist', JSON.stringify(watchlist.value)) } catch {} }
    const watchlistSorted = computed(() => [...watchlist.value].sort((a,b) => (b.ts || 0) - (a.ts || 0)))
    const watchlistPage = ref(1)
    const watchlistTotalPages = computed(() => Math.max(1, Math.ceil(watchlistSorted.value.length / WATCH_PAGE_SIZE)))
    const pagedWatch = computed(() => {
      const start = (watchlistPage.value - 1) * WATCH_PAGE_SIZE
      const end = start + WATCH_PAGE_SIZE
      return watchlistSorted.value.slice(start, end)
    })
    watch(watchlistSorted, () => { watchlistPage.value = Math.min(watchlistPage.value, watchlistTotalPages.value) })
    const clearWatchlist = () => { watchlist.value = []; syncWatchlist(); watchlistPage.value = 1 }

    function wlKey(type, item) {
      const t = String(type || '').toLowerCase()
      if (t === 'fx') return fx:${String(item.symbol || item.key || '').toUpperCase().replace(/[^A-Z0-9]/g,'')}
      if (t === 'crypto') {
        const base = String(item.id || item.symbol || item.key || '').toLowerCase()
        return crypto:${base}
      }
      return ${t}:${String(item.symbol || item.key || item.label || '').toUpperCase()}
    }
    function wlSymbolForMaps(w) {
      if (w.type === 'fx') return String(w.meta?.symbol || w.label).toUpperCase().replace(/[^A-Z0-9]/g,'')
      if (w.type === 'crypto') return String(w.meta?.id || w.meta?.symbol || w.label).toUpperCase()
      const raw = String(w.meta?.symbol || w.label || '').toUpperCase()
      return raw || String((w.key||'').split(':').at(-1)).toUpperCase()
    }
    const inWatchlist = (item, type) => !!watchlist.value.find(w => w.key === wlKey(type, item))
    const toggleWatchlist = (item, type) => {
      const key = wlKey(type, item)
      const idx = watchlist.value.findIndex((w) => w.key === key)
      if (idx >= 0) { watchlist.value.splice(idx, 1) }
      else { watchlist.value.push({ key, label: item.symbol || item.name || item.label, type, meta: item, ts: Date.now() }) }
      syncWatchlist()
    }
    const addToWatchlist = (w) => { const idx = watchlist.value.findIndex((x) => x.key === w.key); if (idx < 0) { watchlist.value.push({ ...w, ts: Date.now() }); syncWatchlist() } }
    const removeFromWatchlist = (w) => { watchlist.value = watchlist.value.filter((x) => x.key !== w.key); syncWatchlist() }
    const isSelectedWatch = (w) => {
      if (String(w.type).toLowerCase() !== String(activeClass.value).toLowerCase()) return false
      return wlSymbolForMaps(w) === String(activeSymbol.value).toUpperCase()
    }

    const mapGet = (obj, key) => (obj?.[key] ?? obj?.[key?.toUpperCase?.()] ?? obj?.[key?.toLowerCase?.()] ?? null)
    const latestPriceMap = ref({})
    const changeMap = ref({})
    function latestPriceForWatch(w){ return mapGet(latestPriceMap.value, wlSymbolForMaps(w)) }
    function latestChangeForWatch(w){ return mapGet(changeMap.value, wlSymbolForMaps(w)) }

    function quickSelect(w) {
      if (!w) return
      const t = String(w.type || '').toLowerCase()
      if (t === 'fx') { selectPair({ symbol: w.meta?.symbol || w.label }, 'fx'); return }
      if (t === 'crypto') {
        selectPair({ id: w.meta?.id || String(w.meta?.symbol || w.label).toLowerCase(), symbol: w.meta?.symbol || w.label, name: w.meta?.name || w.label }, 'crypto')
        return
      }
      activeSymbol.value = wlSymbolForMaps(w)
      activeName.value = w.label
      activeClass.value = t
      refreshChart()
      refreshNews()
    }

    function openFullChartFromWatchNewTab(w) {
      if (!w) return
      const t = String(w.type || '').toLowerCase()
      const key = (t === 'crypto') ? (w.meta?.id || String(w.meta?.symbol || w.label).toLowerCase()) : String(w.meta?.symbol || w.label).toUpperCase()
      openFullChartNewTab({ key, class: t })
    }

    // ===== Helpers =====
    const sanitizeFx = (s) => String(s || '').toUpperCase().replace(/[^A-Z0-9]/g, '')
    const normIndex = (s) => {
      const u = String(s || '').toUpperCase()
      if (/^(?:\^?GSPC|SPX|SPY)$/.test(u)) return 'SPX'
      if (/^(?:\^?NDX|QQQ)$/.test(u)) return 'NDX'
      if (/^(?:\^?DJI|DIA)$/.test(u)) return 'DJI'
      return u
    }
    const num = (v) => { const n = Number(v); return Number.isFinite(n) ? n : NaN }

    // ===== Dynamic pairs =====
    const fxPairs = ref([])
    const cryptoCoins = ref([])
    const pairsLoadedFx = ref(false)
    const pairsLoadedCrypto = ref(false)

    async function loadFxPairs(){
      try {
        const j = await authed('/market/pairs', { params: { asset: 'fx' } })
        const arr = Array.isArray(j) ? j : (j?.items || j?.pairs || [])
        if (arr.length) {
          fxPairs.value = arr
            .map(x => { const s = sanitizeFx(x.symbol || x.ticker || x.key || x.pair || ''); return { symbol: s, icon: /icons/${s}.png } })
            .filter(p => p.symbol)
          pairsLoadedFx.value = fxPairs.value.length > 0
          return
        }
      } catch {}
      try {
        const t = await authed('/market/top', { params: { asset: 'fx', count: 24 } })
        const items = Array.isArray(t) ? t : (t?.items || [])
        if (items.length) {
          fxPairs.value = items.map(r => {
            const s = sanitizeFx(r.symbol || r.ticker || r.key || r.pair || '')
            return { symbol: s, icon: /icons/${s}.png }
          }).filter(p => p.symbol)
          pairsLoadedFx.value = fxPairs.value.length > 0
          return
        }
      } catch {}
      fxPairs.value = [
        { symbol: 'EURUSD', icon: '/icons/EURUSD.png' },
        { symbol: 'GBPUSD', icon: '/icons/GBPUSD.png' },
        { symbol: 'USDJPY', icon: '/icons/USDJPY.png' },
        { symbol: 'XAUUSD', icon: '/icons/XAUUSD.png' },
      ]
      pairsLoadedFx.value = true
    }

    async function loadCryptoPairs(){
      try {
        const j = await authed('/market/pairs', { params: { asset: 'crypto' } })
        const arr = Array.isArray(j) ? j : (j?.items || j?.pairs || j?.coins || [])
        if (arr.length) {
          cryptoCoins.value = arr.map(x => ({
            id: String(x.id || x.symbol || x.key || '').toLowerCase(),
            symbol: String(x.symbol || x.ticker || x.code || '').toUpperCase(),
            name: x.name || x.symbol || x.id || '',
            icon: /icons/crypto/${String(x.symbol || x.ticker || x.code || '').toUpperCase()}.png
          })).filter(c => c.id && c.symbol)
          pairsLoadedCrypto.value = cryptoCoins.value.length > 0
          return
        }
      } catch {}
      try {
        const t = await authed('/market/top', { params: { asset: 'crypto', count: 24 } })
        const items = Array.isArray(t) ? t : (t?.items || [])
        if (items.length) {
          cryptoCoins.value = items.map(r => ({
            id: String(r.id || r.symbol || r.key || '').toLowerCase(),
            symbol: String(r.symbol || r.ticker || r.key || '').toUpperCase(),
            name: r.name || r.symbol || r.id || '',
            icon: /icons/crypto/${String(r.symbol || r.ticker || r.key || '').toUpperCase()}.png
          })).filter(c => c.id && c.symbol)
          pairsLoadedCrypto.value = cryptoCoins.value.length > 0
          return
        }
      } catch {}
      cryptoCoins.value = [
        { id: 'bitcoin',  symbol: 'BTC', name: 'Bitcoin',  icon: '/icons/crypto/BTC.png' },
        { id: 'ethereum', symbol: 'ETH', name: 'Ethereum', icon: '/icons/crypto/ETH.png' },
        { id: 'solana',   symbol: 'SOL', name: 'Solana',   icon: '/icons/crypto/SOL.png' },
      ]
      pairsLoadedCrypto.value = true
    }

    async function ensurePairsLoaded(){ if (!pairsLoadedFx.value) await loadFxPairs(); if (!pairsLoadedCrypto.value) await loadCryptoPairs() }

    const drawerTabs = ref([
      { key:'indices', label:'Indices' },
      { key:'futures', label:'Futures' },
      { key:'stocks',  label:'Stocks'  },
      { key:'etfs',    label:'ETFs'    },
      { key:'bonds',   label:'Bonds'   },
      { key:'fx',      label:'FX'      },
      { key:'crypto',  label:'Crypto'  }
    ])
    const marketTab = ref('indices')

    const activeTab = ref('forex')
    function setTab(t){ activeTab.value = t }

    function selectPair(p, type) {
      if (type === 'fx') {
        const s = sanitizeFx(p.symbol)
        activeSymbol.value = s; activeName.value = s; activeClass.value = 'fx'
      } else if (type === 'crypto') {
        const id = p.id || (p.symbol ? String(p.symbol).toLowerCase() : '')
        activeSymbol.value = id || p.symbol
        activeName.value = p.symbol || p.name || id
        activeClass.value = 'crypto'
      }
      refreshChart()
      refreshNews()
    }

    const iconFor = (asset, symRaw) => {
      const a = String(asset || '').toLowerCase()
      const s = a === 'fx' ? sanitizeFx(symRaw) : (a === 'indices' ? normIndex(symRaw) : String(symRaw || '').toUpperCase())
      if (a === 'fx') return /icons/${s}.png
      if (a === 'crypto') return /icons/crypto/${s}.png
      if (a === 'indices') return /icons/indices/${s}.png
      if (a === 'stocks') return /icons/stocks/${s}.png
      if (a === 'etfs') return /icons/etfs/${s}.png
      if (a === 'bonds') return /icons/bonds/${s}.png
      if (a === 'futures') return /icons/futures/${s}.png
      return /icons/generic/${a || 'asset'}.png
    }
    const fallbackFor = (asset) => /icons/generic/${String(asset||'asset').toLowerCase()}.png
    const onIconError = (e, asset) => {
      if (!e?.target) return
      const cur = e.target.getAttribute('data-fb')
      if (cur) { e.target.style.display='none'; return }
      e.target.src = fallbackFor(asset)
      e.target.setAttribute('data-fb', '1')
    }

    function normalizeAssetParam(asset) {
      const m = {
        index:'indices', indices:'indices',
        stock:'stocks', stocks:'stocks', equity:'stocks',
        etf:'etfs', etfs:'etfs',
        bond:'bonds', bonds:'bonds', treasury:'bonds',
        future:'futures', futures:'futures',
        forex:'fx', fx:'fx',
        crypto:'crypto'
      }
      const key = String(asset || '').toLowerCase()
      return m[key] || key
    }

    function resolveApiTarget(asset, symbol){
      const fam = normalizeAssetParam(asset)
      if (fam === 'crypto') {
        return { asset: 'crypto', symbol: String(symbol || '').toLowerCase(), id: String(symbol || '').toLowerCase() }
      }
      if (fam === 'fx') {
        return { asset: 'fx', symbol: sanitizeFx(symbol) }
      }
      const symU = String(symbol || '').toUpperCase()
      if (fam === 'indices') {
        if (/^(?:\^?GSPC|SPX|SPY)$/.test(symU)) return { asset: 'etfs', symbol: 'SPY', proxyOf: 'SPX' }
        if (/^(?:\^?NDX|QQQ)$/.test(symU)) return { asset: 'etfs', symbol: 'QQQ', proxyOf: 'NDX' }
        if (/^(?:\^?DJI|DIA)$/.test(symU)) return { asset: 'etfs', symbol: 'DIA', proxyOf: 'DJI' }
      }
      return { asset: fam, symbol: symU }
    }

    const getFetchParams = () => resolveApiTarget(activeClass.value, activeSymbol.value)
    function supportsQuote(asset){ const fam = normalizeAssetParam(asset); return fam === 'stocks' || fam === 'etfs' }

    function fromTopCache(asset, symbol){
      const fam = normalizeAssetParam(asset)
      const symU = String(symbol).toUpperCase()
      const row = (topByClass[fam] || []).find(x => x.symbol === symU || x.key === symU || x.label === symU)
      return row ? { price: row.price, change: row.change } : null
    }

    async function ohlcFallback(symbol, interval, asset, signal){
      try{
        const { asset: fam, symbol: sym } = resolveApiTarget(asset || activeClass.value, symbol)
        const k = await authed('/market/ohlc', { params: { asset: fam, symbol: sym, tf: interval, interval }, signal })
        const arr = Array.isArray(k) ? k : (k?.candles || [])
        const n = arr.length
        const last = arr[n - 1], prev = arr[n - 2]
        if (!last) return { price:null, change:null }
        const price = Number(last.close)
        const ch = (prev && Number(prev.close)) ? ((price - Number(prev.close)) / Number(prev.close)) * 100 : 0
        return { price, change: ch }
      } catch { return { price:null, change:null } }
    }

    // Markets / Top lists
    const topByClass = reactive({ indices:[], stocks:[], etfs:[], bonds:[], futures:[], fx:[], crypto:[] })
    function coerceTopItem(raw, cls) {
      let symbol = raw.symbol ?? raw.ticker ?? raw.pair ?? raw.sym ?? raw.id ?? raw.key ?? raw.name ?? raw.code ?? ''
      let displaySym
      if (cls === 'fx') displaySym = sanitizeFx(symbol)
      else if (cls === 'indices') displaySym = normIndex(symbol)
      else displaySym = String(symbol || '').toUpperCase()

      let price = num(raw.price ?? raw.last ?? raw.close ?? raw.value ?? raw.rate ?? raw.mid ?? raw.mid_price ?? raw.lp ?? raw.c ?? raw.p)
      if (!Number.isFinite(price)) {
        const bid = num(raw.bid), ask = num(raw.ask)
        if (Number.isFinite(bid) && Number.isFinite(ask)) price = (bid + ask) / 2
      }

      let ch = num(raw.change ?? raw.change_pct ?? raw.chg ?? raw.dp ?? raw.percent ?? raw.pct ?? raw.changePercent ?? raw.change_percent)
      if (!Number.isFinite(ch)) {
        const prev = num(raw.prevClose ?? raw.pc ?? raw.previous_close ?? raw.prev ?? raw.prior)
        if (Number.isFinite(prev) && Number.isFinite(price) && prev !== 0) {
          ch = ((price - prev) / prev) * 100
        } else {
          const open = num(raw.open ?? raw.o)
          if (Number.isFinite(open) && Number.isFinite(price) && open !== 0) {
            ch = ((price - open) / open) * 100
          }
        }
      }
      const id = cls === 'crypto' ? (raw.id || String(displaySym || '').toLowerCase()) : undefined
      return { key: displaySym, label: displaySym, price, change: Number.isFinite(ch) ? ch : 0, class: cls, asset: cls, symbol: displaySym, id, priceNum: price, icon: iconFor(cls, displaySym) }
    }

    const DEFAULT_LIST = {
      indices: ['SPX','NDX','DJI'],
      futures: ['ES1!','NQ1!','CL1!'],
      stocks:  ['AAPL','MSFT','NVDA'],
      etfs:    ['SPY','QQQ','IWM'],
      bonds:   ['TLT','IEF','HYG'],
      fx:      ['EURUSD','GBPUSD','USDJPY','XAUUSD'],
      crypto:  ['BTCUSDT','ETHUSDT','SOLUSDT'],
    }
    const HYDRATE_TF = '1h'
    function needsHydrate(it){ return !(Number.isFinite(it?.price) && Number.isFinite(it?.change)) }

    async function fetchOhlcHydrate(cls, sym){
      const target = resolveApiTarget(cls, sym)
      try {
        const d = await authed('/market/ohlc', { params: { asset: target.asset, symbol: target.symbol, tf: HYDRATE_TF, interval: HYDRATE_TF } })
        const arr = Array.isArray(d) ? d : (d?.candles || [])
        const n = arr.length
        const last = arr[n - 1]
        const prev = arr[n - 2]
        if (!last) return null
        const price = Number(last.close)
        const change = (prev && Number(prev.close)) ? ((price - Number(prev.close)) / Number(prev.close)) * 100 : 0
        return coerceTopItem({ symbol: sym, price, change, id: cls==='crypto' ? String(sym).toLowerCase() : undefined }, cls)
      } catch { return null }
    }

    async function hydrateSeedsForClass(cls){
      const seeds = DEFAULT_LIST[cls] || []
      const results = await Promise.allSettled(seeds.map((sym) => fetchOhlcHydrate(cls, sym)))
      return results.map(r => r.status === 'fulfilled' ? r.value : null).filter(Boolean)
    }

    async function fillMissingPricesForClass(cls){
      const list = topByClass[cls] || []
      const jobs = list.map(async (it) => {
        if (!needsHydrate(it)) return it
        const hydrated = await fetchOhlcHydrate(cls, it.id || it.symbol || it.key || it.label)
        if (hydrated) { it.price = hydrated.price; it.change = hydrated.change; it.priceNum = hydrated.price }
        return it
      })
      await Promise.allSettled(jobs)
      topByClass[cls] = (topByClass[cls] || []).filter(row => Number.isFinite(row.price))
    }

    // --- Abort controllers & ids
    let chartAbort = null
    const topAborts = Object.create(null)
    let newsAbort = null
    let tilesAbort = null
    const top4Aborts = Object.create(null)
    let chartReqId = 0

    async function loadTopClass(assetKey) {
      const cls = normalizeAssetParam(assetKey)
      try { topAborts[cls]?.abort?.() } catch {}
      topAborts[cls] = new AbortController()
      try {
        const j = await authed('/market/top', { params: { asset: cls, count: 20 }, signal: topAborts[cls].signal })
        if (j?.attribution || j?.source || j?.provider) addProvidersFromText(j.attribution || j.source || j.provider)
        const items = Array.isArray(j) ? j : (j?.items || j?.top || [])
        topByClass[cls] = (items || []).slice(0, 24).map(r => coerceTopItem(r, cls))
      } catch { topByClass[cls] = [] }

      if (!topByClass[cls]?.length) {
        const hydrated = await hydrateSeedsForClass(cls)
        if (hydrated.length) {
          topByClass[cls] = hydrated
        } else {
          const seeds = DEFAULT_LIST[cls] || []
          topByClass[cls] = seeds.map(sym => {
            const s = cls === 'fx' ? sanitizeFx(sym) : (cls === 'indices' ? normIndex(sym) : String(sym).toUpperCase())
            return { key: s, label: s, price: NaN, change: 0, class: cls, asset: cls, symbol: s, id: cls === 'crypto' ? String(s).toLowerCase() : undefined, icon: iconFor(cls, s) }
          })
        }
      }
      if ((topByClass[cls] || []).some(needsHydrate)) { await fillMissingPricesForClass(cls) }
    }

    async function refreshTop() {
      await Promise.allSettled([
        loadTopClass('indices'),
        loadTopClass('futures'),
        loadTopClass('stocks'),
        loadTopClass('etfs'),
        loadTopClass('bonds'),
        loadTopClass('fx'),
        loadTopClass('crypto'),
      ])

      const seed = (cls) => (topByClass[cls] || []).forEach(it => {
        const keyU = String(it.symbol || it.key || it.label).toUpperCase()
        latestPriceMap.value = { ...latestPriceMap.value, [keyU]: it.price, [keyU.toLowerCase()]: it.price }
        changeMap.value = { ...changeMap.value, [keyU]: it.change, [keyU.toLowerCase()]: it.change }
      })
      ;['indices','futures','stocks','etfs','bonds','fx','crypto'].forEach(seed)

      await refreshTopTiles()
      if (marketsView.value === 'top4') await loadTop4ForMarket()
    }

    const topMovers = computed(() => {
      const all = [
        ...topByClass.indices, ...topByClass.futures, ...topByClass.stocks,
        ...topByClass.etfs,    ...topByClass.bonds,  ...topByClass.fx, ...topByClass.crypto
      ]
      const uniq = new Map()
      for (const x of all) { const k = x.asset+':'+x.symbol; if (!uniq.has(k)) uniq.set(k, x) }
      return [...uniq.values()].sort((a,b)=> Math.abs(b.change)-Math.abs(a.change)).slice(0, 50)
    })

    const headerBestByMarket = computed(() => {
      const order = ['fx','crypto','indices','stocks','etfs','bonds']
      const best = (cls) => {
        const arr = (topByClass[cls] || []).filter(r => Number.isFinite(r.change))
        if (!arr.length) return null
        return arr.reduce((a,b)=> (Number(b.change ?? -Infinity) > Number(a.change ?? -Infinity) ? b : a))
      }
      return order.map(cls => {
        const b = best(cls)
        return b ? { asset: cls, symbol: b.symbol, id: b.id, price: b.price, change: b.change } : null
      }).filter(Boolean)
    })

    const headerMovers = computed(() => {
      const primary = headerBestByMarket.value
      if (primary && primary.length) return primary
      return (topMovers.value || []).slice(0, 6).map(x => ({ asset: x.asset, symbol: x.symbol, id: x.id, price: x.price, change: x.change }))
    })

    const moversBox = computed(() => (topMovers.value || []).slice(0, 8))

    const myTopAssets = reactive([
      { asset: 'indices', symbol: 'SPX' },
      { asset: 'indices', symbol: 'NDX' },
      { asset: 'fx',      symbol: 'EURUSD' },
      { asset: 'crypto',  symbol: 'bitcoin' },
    ])

    const quoteMap = reactive(new Map())
    async function fetchQuote(asset, symbol, signal){
      const fam = normalizeAssetParam(asset)
      if (supportsQuote(fam)) {
        try{
          const j = await authed('/market/quote', { params: { asset: fam, symbol }, signal })
          if (j?.attribution || j?.source || j?.provider) addProvidersFromText(j.attribution || j.source || j.provider)
          const price = Number(j?.price ?? j?.last ?? j?.close ?? NaN)
          const change = Number(j?.change ?? NaN)
          quoteMap.set(asset + ':' + symbol, { price: Number.isFinite(price) ? price : null, change: Number.isFinite(change) ? change : null })
          return
        } catch {}
      }
      const cached = fromTopCache(fam, symbol)
      if (cached){ quoteMap.set(asset + ':' + symbol, cached); return }
      const f = await ohlcFallback(symbol, ui.tf, fam, signal)
      quoteMap.set(asset + ':' + symbol, f)
    }

    async function refreshTopTiles(){
      try { tilesAbort?.abort?.() } catch {}
      tilesAbort = new AbortController()
      await Promise.allSettled(myTopAssets.map(t => fetchQuote(t.asset, t.symbol, tilesAbort.signal)))
    }

    function onTopMoverClick(e, m){
      if (e?.altKey || e?.metaKey || e?.ctrlKey) {
        toggleWatchlist({ symbol: m.symbol, id: m.id || String(m.symbol).toLowerCase(), key: m.symbol, label: m.symbol }, m.asset)
      } else {
        selectFromTop({ asset: m.asset, symbol: m.symbol, id: m.id })
      }
    }
    function selectFromTop(t){
      const cls = t.asset
      if (cls === 'fx')        selectPair({ symbol: t.symbol }, 'fx')
      else if (cls === 'crypto') selectPair({ id: t.id || String(t.symbol).toLowerCase(), symbol: String(t.symbol).toUpperCase(), name: t.symbol }, 'crypto')
      else { activeSymbol.value = t.symbol; activeName.value = t.symbol; activeClass.value = cls; refreshChart(); refreshNews() }
    }

    // Top 4
    const top4Market = ref('fx')
    const marketOptions = [
      { key: 'fx', label: 'FX' },
      { key: 'crypto', label: 'Crypto' },
      { key: 'indices', label: 'Indices' },
      { key: 'stocks', label: 'Stocks' },
      { key: 'etfs', label: 'ETFs' },
      { key: 'bonds', label: 'Bonds' },
      { key: 'futures', label: 'Futures' },
    ]
    const top4DetailMap = reactive(new Map())
    function openTop4FromSidebar(){ openTop4InPanel() }
    const top4Visible = computed(() => {
      const cls = top4Market.value
      const ranked = [...(topByClass[cls] || [])].sort((a,b)=> (b.change??0)-(a.change??0)).slice(0,4)
      return ranked.map(r => {
        const det = top4DetailMap.get(cls+':'+r.symbol) || {}
        return { ...r, high: det.high ?? null, low: det.low ?? null, close: Number.isFinite(det.close) ? det.close : r.price }
      })
    })
    async function loadTop4ForMarket(){
      const cls = top4Market.value
      const ranked = [...(topByClass[cls] || [])].sort((a,b)=> (b.change??0)-(a.change??0)).slice(0,4)
      ranked.forEach(r => { try { top4Aborts[cls+':'+r.symbol]?.abort?.() } catch {} })
      await Promise.allSettled(
        ranked.map(async (r) => {
          try {
            const ab = new AbortController()
            top4Aborts[cls+':'+r.symbol] = ab
            const target = resolveApiTarget(cls, r.id || r.symbol)
            const d = await authed('/market/ohlc', { params: { asset: target.asset, symbol: target.symbol, tf: '1d', interval: '1d' }, signal: ab.signal })
            if (d?.attribution || d?.source || d?.provider) addProvidersFromText(d.attribution || d.source || d.provider)
            const arr = Array.isArray(d) ? d : (d?.candles || [])
            const n = arr.length
            const last = arr[n - 1]
            if (last) { top4DetailMap.set(cls+':'+r.symbol, { high: Number(last.high), low: Number(last.low), close: Number(last.close) }) }
          } catch {}
        })
      )
    }
    function pinTop4(it){ toggleWatchlist({ symbol: it.symbol, id: it.id || String(it.symbol).toLowerCase(), label: it.label }, it.asset) }

    const visibleDrawerItems = computed(() => topByClass[marketTab.value] || [])
    function quickSelectFromChip(item) {
      if (!item) return
      const cls = String(item.class || item.asset || '').toLowerCase()
      if (cls === 'fx') selectPair({ symbol: item.label }, 'fx')
      else if (cls === 'crypto') selectPair({ id: item.id || item.key.toLowerCase(), symbol: item.symbol, name: item.label }, 'crypto')
      else { activeSymbol.value = item.key; activeName.value = item.label; activeClass.value = cls; refreshChart(); refreshNews() }
    }

    // Chart data
    const currentCandles = ref([])
    const reloadChart = ref(0)
    const chartKey = computed(() => ${String(activeClass.value).toLowerCase()}::${String(activeSymbol.value).toUpperCase()}::${ui.tf}::${reloadChart.value})

    const visibleCandles = computed(() => {
      if (!currentCandles.value.length) return []
      const n = zoomWindow.value
      const start = Math.max(0, currentCandles.value.length - n - panOffset.value)
      const end = Math.max(start + 10, currentCandles.value.length - panOffset.value)
      return currentCandles.value.slice(start, end)
    })

    const loading = reactive({ price: false })
    const errors = reactive({ price: '' })
    const marketChartType = computed(() => (ui.chartMode === 'line' ? 'area' : 'candles'))

    function normalizeCandles(raw){
      const out = []
      const seen = new Set()
      for (const k of raw || []) {
        let ts = Number(k.time ?? k.t ?? k.timestamp ?? k.date ?? 0)
        if (Number.isFinite(ts) && ts > 1e12) ts = Math.floor(ts / 1000)
        if (!Number.isFinite(ts)) continue
        if (seen.has(ts)) continue
        seen.add(ts)
        out.push({ time: ts, open: Number(k.open ?? k.o ?? 0), high: Number(k.high ?? k.h ?? 0), low: Number(k.low ?? k.l ?? 0), close: Number(k.close ?? k.c ?? 0), volume:Number(k.volume?? k.v ?? 0) })
      }
      out.sort((a,b)=> a.time - b.time)
      return out
    }

    async function loadChart() {
      const reqId = ++chartReqId
      errors.price = ''
      loading.price = true
      currentCandles.value = []
      lastCandle.value = null
      ohlc.ready = false
      panOffset.value = 0
      try {
        try { chartAbort?.abort?.() } catch {}
        chartAbort = new AbortController()
        const { asset, symbol } = getFetchParams()
        const data = await authed('/market/ohlc', { params: { asset, symbol, tf: ui.tf, interval: ui.tf }, signal: chartAbort.signal })
        if (data?.attribution || data?.source || data?.provider) addProvidersFromText(data.attribution || data.source || data.provider)
        const arrRaw = Array.isArray(data) ? data : (data?.candles || [])
        const arr = normalizeCandles(arrRaw)
        if (reqId !== chartReqId) return
        if (!arr.length) {
          errors.price = 'No candles returned for ' + (symbol || activeSymbol.value) + ' (' + ui.tf + ').'
          addProvider('wickwise-mock', true, 'Candles via fallback/seed')
          return
        }
        currentCandles.value = arr
        const lc = arr[arr.length - 1]
        lastCandle.value = lc || null
        if (lc) updateCursorFromCandle(lc)
        serverLiveData.value = true
      } catch (e) {
        if (reqId === chartReqId) {
          errors.price = e?.message || 'Failed to load chart.'
          currentCandles.value = []
          serverLiveData.value = false
        }
      } finally {
        if (reqId === chartReqId) loading.price = false
      }
    }
    function refreshChart(){ reloadChart.value++; loadChart() }
    watch(() => ui.tf, () => refreshChart())
    watch(() => ui.chartMode, () => { reloadChart.value++ })
    watch(() => [ui.logScale, ui.show.volume, ui.barSpacing], () => { reloadChart.value++ })

    // Zoom & pan
    const ZOOM_WINDOWS = [80, 160, 300, 600, 1000]
    const zoomIndex = ref(2)
    const panOffset = ref(0)
    const zoomWindow = computed(() => ZOOM_WINDOWS[Math.min(ZOOM_WINDOWS.length-1, Math.max(0, zoomIndex.value))])
    function zoomIn(){ zoomIndex.value = Math.max(0, zoomIndex.value - 1); ui.barSpacing = Math.min(14, ui.barSpacing + 2) }
    function zoomOut(){ zoomIndex.value = Math.min(ZOOM_WINDOWS.length - 1, zoomIndex.value + 1); ui.barSpacing = Math.max(3, ui.barSpacing - 2) }
    function panLeft(){ panOffset.value = Math.min((currentCandles.value.length - 10), panOffset.value + Math.round(zoomWindow.value * 0.25)) }
    function panRight(){ panOffset.value = Math.max(0, panOffset.value - Math.round(zoomWindow.value * 0.25)) }
    function resetView(){ zoomIndex.value = 2; panOffset.value = 0; ui.barSpacing = 6 }

    function sma(vals, n){ const out=[]; let s=0; for(let i=0;i<vals.length;i++){ s+=vals[i]; if(i>=n)s-=vals[i-n]; out.push(i>=n-1?s/n:NaN) } return out }

    const aiOverview = computed(() => {
      const cs = currentCandles.value || []
      const closes = cs.map(c=>c.close)
      const ready = closes.length >= 50
      if (!ready) return { ready:false }
      const ma20 = sma(closes,20), ma50 = sma(closes,50)
      const last = closes[closes.length - 1], m20 = ma20[ma20.length - 1], m50 = ma50[ma50.length - 1]
      const slope = (ma20[ma20.length - 1] ?? 0) - (ma20[Math.max(0, ma20.length - 4)] ?? ma20[ma20.length - 1] ?? 0)
      const regimeUp = (m20 || 0) > (m50 || 0)
      const range = cs.slice(-20).map(k => Math.max(k.high-k.low,0))
      const avgRange = range.reduce((a,b)=>a+b,0)/Math.max(range.length,1)
      const meanPx = closes.slice(-20).reduce((a,b)=>a+b,0)/Math.max(20,1)
      const volText = avgRange && meanPx ? (avgRange/meanPx > 0.01 ? 'Volatile' : 'Calm') : '—'
      return {
        ready:true,
        title: regimeUp ? 'Uptrend with mild pullbacks' : 'Down/Sideways regime',
        momentum: slope>0?'Momentum ↑':slope<0?'Momentum ↓':'Momentum →',
        momentumClass: slope>0?'up':slope<0?'down':'flat',
        regime: regimeUp ? 'Trend: Up' : 'Trend: Side/Down',
        regimeClass: regimeUp ? 'up' : 'down',
        volatility: volText,
        summary: Price ${regimeUp ? 'above' : 'near/below'} 50SMA; 20SMA is ${slope>0?'rising':'falling'}.,
        bullets:[
          Last close: ${Number.isFinite(last)?formatPrice(last):'—'},
          MA20: ${Number.isFinite(m20)?formatPrice(m20):'—'},
          MA50: ${Number.isFinite(m50)?formatPrice(m50):'—'}
        ],
        zeenoTake: regimeUp ? 'Dips to MA20 may find buyers.' : 'Rallies into MA20/50 may fade.'
      }
    })

    // ===== News =====
    const newsItems = ref([]), newsError = ref(''), newsHintLink = ref(''), newsAttribution = ref('')
    const newsLoading = ref(false)
    const newsLimit = ref(8)

    // PATCH A — tabs + counters + reload (Relevant/Global)
    const newsTab = ref('relevant')

    // Global news state
    const globalNewsItems = ref([])
    const globalNewsLoading = ref(false)
    const globalNewsError = ref('')
    const globalNewsAttribution = ref('')
    const globalNewsLimit = ref(8)

    function prettyTime(ts){ if(!ts) return ''; const d=new Date(ts*(ts>1e12?1:1000)); return d.toLocaleString() }

    function isRelevantNews(n, symU, nameU){
      const inTickers = Array.isArray(n.tickers) && n.tickers.map(t=>String(t).toUpperCase()).includes(symU)
      if (inTickers) return true
      const h = String(n.headline || '').toUpperCase()
      const sum = String(n.summary || '').toUpperCase()
      if (symU && (h.includes(symU) || sum.includes(symU))) return true
      if (nameU && (h.includes(nameU) || sum.includes(nameU))) return true
      return false
    }

    const filteredNewsItems = computed(() => {
      const symU = String(activeClass.value === 'crypto' ? (activeName.value || activeSymbol.value) : activeSymbol.value).toUpperCase()
      const nameU = String(activeName.value || '').toUpperCase()
      if (!newsItems.value?.length) return []
      const rel = newsItems.value.filter(n => isRelevantNews(n, symU, nameU))
      return rel.length ? rel : newsItems.value
    })
    const visibleNews = computed(() => filteredNewsItems.value.slice(0, newsLimit.value))

    async function refreshNews() {
      try { newsAbort?.abort?.() } catch {}
      newsAbort = new AbortController()
      newsLoading.value = true
      newsError.value = ''
      newsItems.value = []
      newsAttribution.value = ''
      newsHintLink.value = ''
      try {
        const fam = normalizeAssetParam(activeClass.value)
        const sym = resolveApiTarget(activeClass.value, activeSymbol.value).symbol
        let data = null
        if (fam === 'stocks' || fam === 'etfs') {
          data = await authed('/market/news/company', { params: { symbol: sym, limit: 25 }, signal: newsAbort.signal })
        } else {
          data = await authed('/market/news/global', { params: { asset: fam, limit: 25 }, signal: newsAbort.signal })
        }
        if (data?.attribution || data?.source || data?.provider || data?.notes) {
          addProvidersFromText(data.attribution || data.source || data.provider || data.notes)
          newsAttribution.value = [data.attribution, data.notes].filter(Boolean).join(' · ')
        }
        const arr = Array.isArray(data) ? data : (data?.news || data?.items || [])
        newsItems.value = (arr || []).map(n => ({
          id: n.id || n.guid || ${n.source || 'src'}:${n.datetime || n.time || Date.now()},
          url: n.url || n.link || '#',
          headline: n.headline || n.title || '(untitled)',
          source: n.source || 'News',
          datetime: Number(n.datetime || n.time || Date.now()/1000),
          tickers: Array.isArray(n.tickers) ? n.tickers : (Array.isArray(n.symbols) ? n.symbols : []),
          summary: n.summary || n.description || ''
        }))
        if (!newsItems.value.length) {
          const q = (fam === 'fx') ? ${activeSymbol.value} forex : (fam === 'crypto' ? ${activeName.value || activeSymbol.value} crypto : activeSymbol.value)
          newsHintLink.value = https://news.google.com/search?q=${encodeURIComponent(q)}
        } else {
          newsHintLink.value = ''
        }
      } catch (e) {
        newsError.value = e?.message || 'Failed to load news'
      } finally {
        newsLoading.value = false
      }
    }

    // PATCH B — dedicated global news loader (by current asset class)
    async function refreshGlobalNews() {
      globalNewsLoading.value = true
      globalNewsError.value = ''
      globalNewsItems.value = []
      globalNewsAttribution.value = ''
      try {
        const fam = normalizeAssetParam(activeClass.value)
        const data = await authed('/market/news/global', { params: { asset: fam, limit: 25 } })
        if (data?.attribution || data?.source || data?.provider || data?.notes) {
          addProvidersFromText(data.attribution || data.source || data.provider || data.notes)
          globalNewsAttribution.value = [data.attribution, data.notes].filter(Boolean).join(' · ')
        }
        const arr = Array.isArray(data) ? data : (data?.news || data?.items || [])
        globalNewsItems.value = (arr || []).map(n => ({
          id: n.id || n.guid || ${n.source || 'src'}:${n.datetime || n.time || Date.now()},
          url: n.url || n.link || '#',
          headline: n.headline || n.title || '(untitled)',
          source: n.source || 'News',
          datetime: Number(n.datetime || n.time || Date.now()/1000),
          tickers: Array.isArray(n.tickers) ? n.tickers : (Array.isArray(n.symbols) ? n.symbols : []),
          summary: n.summary || n.description || ''
        }))
      } catch (e) {
        globalNewsError.value = e?.message || 'Failed to load global news'
      } finally {
        globalNewsLoading.value = false
      }
    }
    const visibleGlobalNews = computed(() => globalNewsItems.value.slice(0, globalNewsLimit.value))

    // PATCH C — convenience reload for both lists
    async function reloadNewsBoth(){
      await Promise.allSettled([ refreshNews(), refreshGlobalNews() ])
    }

    function openFullChartNewTab(it) {
      const sym = it?.key || it?.id || activeSymbol.value
      const type = (it?.class || it?.asset || activeClass.value) || ''
      try {
        const { href } = router?.resolve?.({ name: 'FullChart', query: { symbol: sym, type, popout: '0' } }) || {}
        if (href) { openNamedChartPopout(href, 'WW-Chart') } else { throw new Error('no-router') }
      } catch {
        const qs = new URLSearchParams({ symbol: sym, type, popout: '0' }).toString()
        openNamedChartPopout('/fullchart?' + qs, 'WW-Chart')
      }
    }
    function openFullChartFromCurrentNewTab() {
      const sym = activeSymbol.value
      const type = activeClass.value
      try {
        const { href } = router?.resolve?.({ name: 'FullChart', query: { symbol: sym, type, popout: '0' } }) || {}
        if (href) { openNamedChartPopout(href, 'WW-Chart') } else { throw new Error('no-router') }
      } catch {
        const qs = new URLSearchParams({ symbol: sym, type, popout: '0' }).toString()
        openNamedChartPopout('/fullchart?' + qs, 'WW-Chart')
      }
    }

    function refreshAll(){ refreshTop(); refreshTopTiles(); refreshChart(); refreshNews() }

    function formatPrice(n){
      if(n==null||!Number.isFinite(n)) return '—'
      if(n>=1000) return n.toFixed(2)
      if(n>=100) return n.toFixed(2)
      if(n>=1) return n.toFixed(3)
      return n.toFixed(5)
    }
    function formatVol(v){ const n=Number(v||0); if(n>=1e9) return (n/1e9).toFixed(2)+'B'; if(n>=1e6) return (n/1e6).toFixed(2)+'M'; if(n>=1e3) return (n/1e3).toFixed(2)+'K'; return n.toFixed(0) }
    function signedPct(n){ const v=Number(n|| 0); return ${v>=0?'+':''}${v.toFixed(2)}% }
    function prettyMarket(k){ const m = { fx:'FX', crypto:'Crypto', indices:'Indices', stocks:'Stocks', etfs:'ETFs', bonds:'Bonds', futures:'Futures' }; return m[k] || k }

    async function logout(){
      try { await signOut(auth) } catch {}
      try { completeLogout() } catch {}
      await safePush({ name: 'Home' }).catch(()=>{})
      await safePush('/').catch(()=>{})
    }

    function closeAllTransientUI(){
      chartMenuOpen.value = false
      if (rightOverlay.open) rightOverlay.open = false
      if (settingsOpen.value) settingsOpen.value = false
      if (marketPanel.open) marketPanel.open = false
    }

    function uuid4() {
      try {
        const g = (typeof globalThis !== 'undefined' && globalThis.crypto) || (typeof window !== 'undefined' && window.crypto)
        if (g && g.getRandomValues) {
          const a = new Uint8Array(16); g.getRandomValues(a)
          a[6] = (a[6] & 0x0f) | 0x40
          a[8] = (a[8] & 0x3f) | 0x80
          const toHex = (n) => n.toString(16).padStart(2, '0')
          const s = Array.from(a, toHex).join('')
          return ${s.slice(0,8)}-${s.slice(8,12)}-${s.slice(12,16)}-${s.slice(16,20)}-${s.slice(20)}
        }
      } catch {}
      return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, c => {
        const r = Math.random() * 16 | 0
        const v = c === 'x' ? r : (r & 0x3) | 0x8
        return v.toString(16)
      })
    }
    const userId = ref(localStorage.getItem('userId') || (localStorage.setItem('userId', uuid4()), localStorage.getItem('userId')))

    function prefetchIcons() {
      const head = document.head || document.getElementsByTagName('head')[0]
      const hrefs = [
        '/icons/EURUSD.png','/icons/GBPUSD.png','/icons/USDJPY.png','/icons/XAUUSD.png',
        '/icons/crypto/BTC.png','/icons/crypto/ETH.png','/icons/crypto/SOL.png'
      ]
      hrefs.forEach(h => { try { const l = document.createElement('link'); l.rel = 'prefetch'; l.as = 'image'; l.href = h; head.appendChild(l) } catch {} })
    }

    function onOpenPricing(e){ const plan = e?.detail?.plan || 'Gold'; console.log('[Pricing] open plan:', plan) }

    const serviceNotice = computed(() => { if (serverAuthError.value) return serverAuthError.value; return '' })

    const zeenoProps = computed(() => ({
      apiKey: marketApiKey,
      selectedPair: { symbol: activeSymbol.value, name: activeName.value || activeSymbol.value },
      timeframe: ui.tf,
      candles: currentCandles.value,
      lastCandle: lastCandle.value,
      subscriptionTier: devPlanLabel.value,
      apiBase,
      userId: userId.value,
      serverKey: backendKey || ''
    }))

    function openZeeno(evt){
      if (evt?.shiftKey || evt?.ctrlKey || evt?.metaKey) { openZeenoNewTab(); return }
      try { localStorage.setItem('zeenoMin','false'); localStorage.setItem('wickbotMin','false') } catch {}
      try { window.dispatchEvent(new Event('zeeno:open')) } catch {}
      closeAllTransientUI()
    }

    const _zeenoEnterEvts = ['zeeno:enlarge','zeeno:expand','zeeno:open-full','zeeno:fullscreen','zeeno:enter-fullscreen','zeeno:open-tab']
    function _zeenoOpenInNewTab (e) { try { e?.preventDefault?.() } catch {}; try { e?.stopPropagation?.() } catch {}; openZeenoNewTab() }

    const rootClasses = computed(() => ({ light: theme.value === 'light', 'sidebar-collapsed': sidebarCollapsed.value }))

    function onZeenoPopinRequest(e){
      try {
        const msg = e?.data
        if (!msg || msg.type !== 'ZEENO_POPIN_REQUEST') return
        if (e.origin !== window.location.origin) return
        const sid = msg.sid || ''
        const payload = buildZeenoPopoutPayload()
        persistZeenoPopoutPayload(sid, payload)
        try { e.source?.postMessage({ type: 'ZEENO_POPIN_PAYLOAD', sid, payload }, e.origin) } catch {}
      } catch {}
    }

    let timer=null
    let unguard = null
    function onKey(e){
      if (e.key === 'Escape') {
        if (rightOverlay.open) closeOverlay()
        if (marketPanel.open) closeMarketsPanel()
        if (chartMenuOpen.value) chartMenuOpen.value = false
        if (settingsOpen.value) settingsOpen.value = false
      }
    }
    function tick(){ try { if (document.hidden) return } catch {}; refreshAll() }

    onMounted(async ()=>{
      initThemeSync()
      try { document.documentElement.setAttribute('data-theme', theme.value) } catch {}
      try { window.dispatchEvent(new Event('themechange')) } catch {}
      if (!IS_TEST) { try { await ensureAuth() } catch {} }
      fetchDevTier()
      probeProviders()
      prefetchIcons()
      refreshAll()
      timer=setInterval(tick, autoRefreshMs)
      window.addEventListener('keydown', onKey)
      window.addEventListener('resize', onResize)
      window.addEventListener('ww:crosshair', onGlobalCrosshair)
      window.addEventListener('open-pricing', onOpenPricing)
      window.addEventListener('dashboard:open-settings', onOpenSettingsBridge)
      _zeenoEnterEvts.forEach(n => window.addEventListener(n, _zeenoOpenInNewTab, { passive: false }))
      try { localStorage.setItem('zeenoMin','false'); localStorage.setItem('wickbotMin','false') } catch {}
      try { window.dispatchEvent(new CustomEvent('zeeno:open')) } catch {}

      // Block in-tab Zeeno/FullChart routes & redirect to popouts
      if (router && typeof router.beforeEach === 'function') {
        unguard = router.beforeEach((to, from, next) => {
          const targetName = String(to?.name || '')
          if (targetName === 'ZeenoPopout') { openZeenoNewTab(); next(false); return }
          if (targetName === 'FullChart' || targetName === 'FullChartDirect') {
            const q = to?.query || {}
            try {
              const { href } = router.resolve({ name: 'FullChart', query: q })
              openNamedChartPopout(href, 'WW-Chart')
            } catch { openFullChartFromCurrentNewTab() }
            next(false); return
          }
          next()
        })
      }
      window.addEventListener('message', onZeenoPopinRequest)
    })

    onBeforeUnmount(()=>{
      if(timer) clearInterval(timer)
      window.removeEventListener('keydown', onKey)
      window.removeEventListener('resize', onResize)
      window.removeEventListener('ww:crosshair', onGlobalCrosshair)
      window.removeEventListener('open-pricing', onOpenPricing)
      window.removeEventListener('dashboard:open-settings', onOpenSettingsBridge)
      _zeenoEnterEvts.forEach(n => window.removeEventListener(n, _zeenoOpenInNewTab))
      try { disconnectThemeObserver?.() } catch {}
      if (raf) cancelAnimationFrame(raf)
      try { chartAbort?.abort?.() } catch {}
      try { newsAbort?.abort?.() } catch {}
      try { tilesAbort?.abort?.() } catch {}
      try { Object.values(topAborts).forEach(a => a?.abort?.()) } catch {}
      try { Object.values(top4Aborts).forEach(a => a?.abort?.()) } catch {}
      try { unguard?.() } catch {}
      window.removeEventListener('message', onZeenoPopinRequest)
    })

    // Keep news in sync with selection
    watch([activeSymbol, activeClass], async () => {
      await refreshNews()
      // Refresh global tab too if user is viewing it, so the counter stays accurate
      if (newsTab.value === 'global') await refreshGlobalNews()
    })

    const changeForSelected = computed(() => {
      const fam = normalizeAssetParam(activeClass.value)
      const key = fam === 'crypto' ? String(activeSymbol.value).toLowerCase() : String(activeSymbol.value).toUpperCase()
      return changeMap.value[key]
    })

    return {
      // state / refs
      fatalError: fatalErrorRef,
      router,
      wickwiseEmblem,
      tvEnabled,
      isAuthenticated,
      theme, toggleTheme, setTheme, instantToggleTheme,
      sidebarCollapsed, persistSidebar,
      leftTab, rightOverlay, openOverlay, closeOverlay, overlayEl,
      settingsOpen, toggleSettings, settingsEl,
      chartMenuOpen, toggleChartTypeMenu,
      ui, setChartMode,
      activeSymbol, activeName, activeClass, selectedLabel,
      ohlc, updateCursorFromCandle,
      watchlist, watchlistSorted, clearWatchlist, inWatchlist, toggleWatchlist, addToWatchlist, removeFromWatchlist, isSelectedWatch,
      quickSelect, openFullChartFromWatchNewTab,
      watchlistPage, watchlistTotalPages, pagedWatch,
      latestPriceForWatch, latestChangeForWatch,
      fxPairs, cryptoCoins, drawerTabs, marketTab, visibleDrawerItems, setTab,
      topMovers, headerBestByMarket, headerMovers, selectFromTop, refreshTop, refreshTopTiles, onTopMoverClick, moversBox,
      currentCandles, visibleCandles, lastCandle, reloadChart, loading, errors, refreshChart, chartKey,
      zoomIn, zoomOut, panLeft, panRight, resetView,
      aiOverview,

      // News (Relevant + Global) — includes A/B/C patches
      newsTab,
      newsItems, newsError, newsHintLink, newsAttribution, newsLoading, prettyTime, refreshNews, filteredNewsItems, visibleNews, newsLimit,
      globalNewsItems, globalNewsLoading, globalNewsError, globalNewsAttribution, refreshGlobalNews, visibleGlobalNews, globalNewsLimit,
      reloadNewsBoth,

      refreshAll, openFullChartNewTab, openFullChartFromCurrentNewTab, openHomeInNewTab, openZeenoNewTab,
      formatPrice, formatVol, signedPct, prettyMarket, iconFor, onIconError,
      top4Market, marketOptions, openTop4FromSidebar, top4Visible, loadTop4ForMarket, pinTop4,
      closeAllTransientUI, logout, quickSelectFromChip,
      marketsView, apiBase, marketApiKey, userId, getFetchParams, resolveApiTarget, normalizeAssetParam,
      isCompact, marketPanel, marketEl, openMarketsPanel, closeMarketsPanel,
      serviceNotice, devPlan, devLevel, devPlanLabel,
      zeenoProps, openZeeno,
      rootClasses, serverLiveData, providerChips, onProviderIconError,
      changeForSelected,
      _setFatal, ensurePairsLoaded,
      marketChartType,

      // handlers for ChartCard events
      onCrosshair,
      ingestProviders,
      addProvidersFromText,
    }
  }
}
</script>

<style scoped>
/* ====== Layout ====== */
.dashboard-root {
  display: grid;
  grid-template-columns: minmax(190px, 220px) minmax(740px, 1fr) minmax(300px, 360px);
  gap: 12px;
  min-height: 100vh;
  background: var(--bg, #0f1419);
  color: #e9eef2;
}
.dashboard-root.light { --bg:#f6f8fb; color:#1b2430; }
.dashboard-root.sidebar-collapsed { grid-template-columns: 64px minmax(720px, 1fr) minmax(300px, 360px); }

@media (max-width: 1340px){
  .dashboard-root { grid-template-columns: minmax(64px, 220px) 1fr; grid-template-areas: "sidebar main" "sidebar right"; }
  .sidebar { grid-area: sidebar; }
  .main { grid-area: main; }
  .right-rail { grid-area: right; }
}
@media (max-width: 1080px){
  .dashboard-root { grid-template-columns: 64px 1fr; }
  .sidebar { width: 64px; }
  .sidebar-scroll { display:none; }
}
@media (max-width: 1180px){ .chart-block { height: 460px; } }

.app-error{
  position: fixed; left: 12px; bottom: 12px; z-index: 1000;
  border:1px solid rgba(255,80,80,.35);
  background: rgba(255,80,80,.12);
  color:#ffd3d3; border-radius:10px; padding:8px 10px; font-size:.9rem; backdrop-filter: blur(6px);
}

.main { display:flex; flex-direction:column; gap:12px; min-height: 0; overflow: auto; min-width: 0; }
.service-notice{ border:1px solid rgba(255,255,255,.18); background:rgba(255,255,255,.06); color:inherit; border-radius:10px; padding:8px 10px; font-weight:700; }

/* ====== Powered by bar ====== */
.powerbar{
  display:flex; align-items:center; gap:10px; flex-wrap:wrap;
  border:1px solid rgba(255,255,255,.12); background:rgba(255,255,255,.04);
  border-radius:12px; padding:8px 10px;
}
.pb-title{ font-weight:900; opacity:.9; font-size:12px; }
.pb-list{ display:flex; gap:6px; align-items:center; list-style:none; margin:0; padding:0; flex-wrap:wrap; }
.pb-chip{
  display:flex; align-items:center; gap:6px;
  border:1px solid rgba(255,255,255,.18);
  background: rgba(255,255,255,.06);
  border-radius:999px; padding:4px 8px; font-size:12px; line-height:1; font-weight:800;
}
.pb-chip.off{ opacity:.7 }
.pb-ic{ width:14px; height:14px; border-radius:4px; object-fit:contain; }
.pb-name{ white-space:nowrap; }
.pb-dot{ display:inline-block; width:6px; height:6px; border-radius:50%; background: rgba(255,255,255,.5); }
.pb-dot.ok{ background:#26a69a } .pb-dot.warn{ background:#efb05e }
.pb-pill{ margin-left:auto; font-size:12px; font-weight:900; border:1px solid rgba(255,255,255,.22); background:linear-gradient(180deg, rgba(255,255,255,.08), rgba(255,255,255,.04)); border-radius:999px; padding:4px 8px; }
.pb-pill.warn{ border-color: rgba(255,120,120,.28); }

/* ====== Settings panel (drop from top) ===== */
.settings-panel{
  border:1px solid rgba(255,255,255,.14); background: rgba(17,24,39,.86);
  backdrop-filter: blur(8px); border-radius: 12px; padding: 10px;
  box-shadow: var(--ww-shadow-2, 0 12px 32px rgba(0,0,0,.45));
}
.dashboard-root.light .settings-panel{ background: rgba(255,255,255,.92); }
.sp-head{ display:flex; align-items:center; justify-content:space-between; margin-bottom:8px; }
.sp-title{ font-weight:900; margin:0; }
.sp-actions{ display:flex; gap:8px; }
.sp-body{ max-height: 70vh; overflow:auto; }
.settings-drop-enter-active, .settings-drop-leave-active { transition: all .18s ease; }
.settings-drop-enter-from, .settings-drop-leave-to { opacity:0; transform: translateY(-10px); }

/* ====== Sidebar ====== */
.sidebar { align-self: start; max-height: 100vh; border:1px solid rgba(255,255,255,0.12); background: rgba(255,255,255,0.04); display: grid; grid-template-rows: auto 1fr; border-radius: 12px; overflow: hidden; }
.sidebar.collapsed { width: 64px; }
.sidebar-top { display:flex; align-items:center; justify-content:space-between; padding:10px; border-bottom:1px solid rgba(255,255,255,.12); }
.brand { display:flex; gap:8px; align-items:center; cursor:pointer; }
.brand-emblem { width:24px; height:24px; }
.brand-text { font-weight: 900; }
.sidebar-scroll { overflow:auto; padding:10px; display:grid; gap:12px; }
.side-tabs { display:flex; flex-direction:column; gap:8px; position:relative; }
.side-tabs button, .side-tab-btn{ display:flex; align-items:center; gap:8px; height:36px; padding:0 10px; border:1px solid rgba(255,255,255,.18); border-radius:999px; background:rgba(255,255,255,.06); color:inherit; cursor:pointer; font-weight:700; font-size:13px; }
.side-tabs button.active, .side-tab-btn.active{ background:rgba(255,255,255,.14) }
.caret{ margin-left:auto; opacity:.7; font-size:11px; }
.tab-with-menu{ position:relative; }
.tab-menu{ position:absolute; top:42px; left:0; z-index:20; background:#111827; border:1px solid rgba(255,255,255,.14); border-radius:10px; padding:6px; display:flex; flex-direction:column; gap:4px; min-width:180px; }
.dashboard-root.light .tab-menu{ background:#ffffff; }
.tab-menu-item{ text-align:left; border:none; background:transparent; color:inherit; padding:6px 8px; border-radius:8px; cursor:pointer; font-weight:700; font-size:12px; }
.tab-menu-item.sel{ background:rgba(255,255,255,.15) }

/* ====== Header ====== */
.topbar{
  position: relative; display:grid; grid-template-columns: 1fr; align-items:center; gap:10px;
  border:1px solid rgba(255,255,255,.12); background:rgba(255,255,255,.04);
  border-radius:12px; padding:6px 8px;
}
.mini-movers{ display:flex; gap:6px; align-items:center; flex-wrap:wrap; }
.mini-title{ opacity:.8; font-weight:800; margin-right:4px; font-size:12px; }
.mini-chip{
  display:flex; gap:4px; align-items:center; border:1px solid rgba(255,255,255,.18); background: transparent;
  border-radius:9999px; padding:2px 6px; cursor:pointer; font-size:11.5px; line-height:1.1;
  transition: background .12s ease, border-color .12s ease, transform .04s ease;
}
.mini-chip:hover{ background:rgba(255,255,255,.07); border-color:rgba(255,255,255,.24) }
.mini-chip:active{ transform: translateY(1px) }
.mini-ic{ width:12px; height:12px; border-radius:4px; opacity:1 }
.mini-market{ font-weight:800; opacity:.8; }
.mini-symbol{ font-weight:900; opacity:.95 }
.mini-chg.up{ color:#26a69a } .mini-chg.down{ color:#ef5350 }
.mini-hint{ opacity:.7; font-size:12px }

/* ====== MARKETS DROP PANEL ====== */
.markets-panel{
  border:1px solid rgba(255,255,255,.14); background: rgba(17,24,39,.86);
  backdrop-filter: blur(8px); border-radius: 12px; padding: 10px;
  box-shadow: var(--ww-shadow-2, 0 12px 32px rgba(0,0,0,.45));
}
.dashboard-root.light .markets-panel{ background: rgba(255,255,255,.96); }
.mp-head{ display:flex; align-items:center; justify-content:space-between; gap:10px; margin-bottom:8px; }
.mp-title{ font-weight:900; margin:0; }
.mp-actions{ display:flex; align-items:center; gap:8px; flex-wrap:wrap; }
.mp-body{ max-height: 64vh; overflow: hidden; }
.mp-overview{ border:1px solid rgba(255,255,255,.14); border-radius:10px; overflow:hidden; background: rgba(0,0,0,.06); }
.dashboard-root.light .mp-overview{ background: rgba(0,0,0,.03); }
.mp-list{ border:1px solid rgba(255,255,255,.14); border-radius:10px; padding:8px; background: rgba(0,0,0,.06); max-height: 64vh; }
.dashboard-root.light .mp-list{ background: rgba(0,0,0,.03); }
.mp-list-inner{ min-width: 760px; }
.markets-drop-enter-active, .markets-drop-leave-active { transition: all .18s ease; }
.markets-drop-enter-from, .markets-drop-leave-to { opacity:0; transform: translateY(-10px); }

/* ====== KPI ====== */
.section{ margin-top:0; }
.kpi-row{ display:grid; grid-template-columns: 1fr 2fr 1fr; gap:8px; }
@media (max-width: 900px){ .kpi-row{ grid-template-columns: 1fr; } }
.kpi-card{ border:1px solid rgba(255,255,255,.12); background:linear-gradient(180deg, rgba(255,255,255,.05), rgba(255,255,255,.03)); border-radius:12px; padding:10px; }
.kpi-card.highlight{ background:linear-gradient(180deg, rgba(255,255,255,.08), rgba(255,255,255,.03)); }
.kpi-title{ font-weight:900; margin-bottom:6px; }
.kpi-selected{ display:flex; align-items:center; justify-content:space-between; }
.kpi-value{ display:flex; align-items:center; gap:8px; font-weight:900; }
.title-ic{ width:14px; height:14px; border-radius:6px; }

/* ====== Chart area ====== */
.chart-section { display:flex; flex-direction:column; }
.chart-controls{
  display:flex; align-items:center; gap:10px; flex-wrap:wrap;
  border:1px solid rgba(255,255,255,.12); background:rgba(255,255,255,.04);
  border-radius:12px; padding:8px 10px; margin:8px 0;
}
.toolbar{ display:flex; gap:6px; }
.cc-select{ padding:6px 8px; }
.cc-check{ display:flex; align-items:center; gap:6px; }
.spacer{ flex:1; }
.chart-block{
  position: relative; border:1px solid rgba(255,255,255,.12);
  background:rgba(255,255,255,.04); border-radius:12px; height: 520px;
  overflow:hidden; min-width: 0;
}
.chart-shell{ position:relative; height:100%; display:flex; }
.chart-shell > * { flex: 1 1 auto; min-width: 0; min-height: 0; }

/* Hide legacy chrome (no-op now) */
.chart-block :deep(.chartcard-head),
.chart-block :deep(.chart-header),
.chart-block :deep(.widget-head),
.chart-block :deep(.chart-settings),
.chart-block :deep(.chart-actions),
.chart-block :deep(.chart-actions .btn),
.chart-block :deep(button[title="Settings"]) {
  display: none !important;
}

/* Overlay buttons on chart */
.chart-overlay{
  position:absolute; top:8px; right:8px; z-index:5; display:flex; gap:6px;
}
.btn.micro{ padding:4px 8px; font-size:.78rem; border-radius:8px; border:1px solid rgba(255,255,255,.18); background:rgba(255,255,255,.06); color:inherit; cursor:pointer; }
.btn.micro.ghost{ background:transparent; border-color:rgba(255,255,255,.22) }
.btn.micro:hover{ background:rgba(255,255,255,.1) }

/* Chart inline error */
.chart-error{
  position:absolute; left:8px; bottom:8px; z-index:6;
  border:1px solid rgba(255,80,80,.35); background: rgba(255,80,80,.12);
  color:#ffd3d3; border-radius:8px; padding:6px 8px; font-size:.85rem;
}

/* OHLC under chart */
.ohlc-card{ border:1px solid rgba(255,255,255,.12); background:rgba(255,255,255,.04); border-radius:12px; padding:8px 10px; margin-top:8px; }
.ohlc-row{ display:grid; grid-template-columns: repeat(6, 1fr); gap:8px; }
@media (max-width: 900px){ .ohlc-row{ grid-template-columns: repeat(3, 1fr); } }
.ohlc-item{ display:flex; flex-direction:column; gap:2px; }
.ohlc-item span{ font-size:12px; opacity:.75; }
.ohlc-item strong{ font-weight:900; }
.ohlc-item strong.up{ color:#26a69a } .ohlc-item strong.down{ color:#ef5350 }

/* ====== Right rail ====== */
.right-rail { display:grid; align-content:start; gap:12px; min-width: 300px; }
.news-card.rail{ border:1px solid rgba(255,255,255,.12); background:linear-gradient(180deg, rgba(255,255,255,.05), rgba(255,255,255,.03)); border-radius:12px; padding:8px; }
.dashboard-root.light .news-card.rail{ background:linear-gradient(180deg, rgba(0,0,0,.03), rgba(0,0,0,.01)); }
.rail-head{ display:flex; align-items:center; justify-content:space-between; margin-bottom:6px; }
.section-title.small{ font-size: 0.95rem; }
.news-list.rail{ list-style:none; padding:0; margin:0; display:grid; gap:6px; }
.news-item.rail{
  display:grid; gap:4px; padding:6px 8px;
  border:1px solid rgba(255,255,255,.06); background: rgba(255,255,255,.02);
  border-radius:10px; transition: background .12s ease, border-color .12s ease, transform .04s ease;
}
.dashboard-root.light .news-item.rail{ background: rgba(0,0,0,.02); border-color: rgba(0,0,0,.06); }
.news-item.rail:hover{ background:rgba(255,255,255,.06); border-color:rgba(255,255,255,.14) }
.dashboard-root.light .news-item.rail:hover{ background:rgba(0,0,0,.06); border-color:rgba(0,0,0,.12) }
.news-link{ color:inherit; text-decoration:none; font-weight:800; line-height:1.25; font-size:.92rem; }
.clamp-2{ display:-webkit-box; -webkit-line-clamp:2; -webkit-box-orient:vertical; overflow:hidden; }
.meta-row{ display:flex; align-items:center; gap:6px; flex-wrap:wrap; font-size:12px; opacity:.9; }
.chip{ display:inline-flex; align-items:center; gap:6px; border:1px solid rgba(255,255,255,.18); background: rgba(255,255,255,.06); padding:2px 6px; border-radius:999px; font-weight:800; line-height:1; }
.chip.ghost{ background: transparent; border-color: rgba(255,255,255,.18); opacity:.9; }
.dashboard-root.light .chip{ background: rgba(0,0,0,.06); border-color: rgba(0,0,0,.14) }
.dashboard-root.light .chip.ghost{ background: transparent; border-color: rgba(0,0,0,.14) }
.news-error{ color:#ef5350; }
.news-attr{ margin-top:6px; opacity:.7; font-size:12px; }
.news-empty{ font-size:.88rem; opacity:.85; padding:6px 2px; }
.news-hint a{ text-decoration:underline; }
.news-more{ margin-top:6px; display:flex; justify-content:flex-end; }

/* Skeleton shimmer */
.skeleton-row{ height: 14px; border-radius:6px; background: linear-gradient(90deg, rgba(255,255,255,.12), rgba(255,255,255,.06), rgba(255,255,255,.12)); background-size:200% 100%; animation: shimmer 1.2s infinite linear; }
.skeleton-chip{ width: 72px; height: 18px; border-radius:999px; display:inline-block; }
@keyframes shimmer{ 0%{ background-position: 200% 0; } 100%{ background-position: -200% 0; } }

/* Watchlist */
.watch-card{ border:1px solid rgba(255,255,255,0.12); background: rgba(255,255,255,0.04); border-radius: 12px; padding: 8px; max-height: 48vh; overflow:auto; }
.empty.small{ font-size: 0.85rem; opacity:.8; }
.watch{ list-style:none; margin:0; padding:0; display:grid; }
.watch.compact{ gap:2px; }
.watch-row{
  display:grid; grid-template-columns: auto 1fr auto auto auto auto; gap:6px; align-items:center;
  padding:6px 8px; border-bottom:1px solid rgba(255,255,255,.08); background:transparent; cursor:pointer;
}
.watch-row:last-child{ border-bottom:none; }

.row-ic{
  width:12px; height:12px; border-radius:9999px; padding:2px 6px;
  background: var(--pill-bg); border:1px solid var(--pill-border); box-shadow: inset 0 0 0 1px var(--pill-glow);
}

.row-name{ font-weight:800; }
.row-price{ opacity:.9; }
.row-chg.up{ color:#26a69a } .row-chg.down{ color:#ef5350 }

.icon-btn.pin{
  width:28px; height:28px; border-radius:999px; border:1px solid rgba(255,255,255,.18);
  background: rgba(255,255,255,.05); display:grid; place-items:center; cursor:pointer;
  transition: filter .15s ease, background .15s ease, border-color .15s ease, transform .15s ease;
}
.icon-btn.pin .ic{ width:14px; height:14px; fill: currentColor; opacity: .9 }
.icon-btn.pin:hover{ background:rgba(255,255,255,.1); border-color:rgba(255,255,255,.28) }
.icon-btn.pin:active{ transform: translateY(1px) }
.icon-btn.pin.active{ background: linear-gradient(180deg, rgba(255,255,255,.18), rgba(255,255,255,.05)); border-color: rgba(255,255,255,.34) }

.wl-x{ border:1px solid rgba(255,255,255,.2); background:rgba(255,255,255,.06); color:inherit; border-radius:8px; padding:4px 6px; cursor:pointer; }

.pager{ display:flex; justify-content:center; align-items:center; gap:8px; }
.pager.compact{ margin-top:6px; }
.pg-btn{ border:1px solid rgba(255,255,255,.2); background:rgba(255,255,255,.06); color:inherit; border-radius:8px; padding:4px 8px; cursor:pointer; }
.pg-ind{ opacity:.85; font-weight:700; font-size:0.9rem; }

/* Right overlay */
.right-overlay{
  position: sticky; top:0; z-index: 10;
  border:1px solid rgba(255,255,255,.14); background: rgba(17,24,39,.72); backdrop-filter: blur(8px);
  border-radius: 12px; padding: 10px; display: grid; grid-template-rows: auto 1fr; outline: none;
  max-height: 80vh; overflow:auto;
}
.right-overlay.floating{
  position: fixed; top: 12px; right: 12px; left: auto; width: min(720px, 96vw);
  max-height: min(90vh, 900px); z-index: 50; box-shadow: 0 20px 60px rgba(0,0,0,.5);
}
.right-overlay .ro-head{ position: sticky; top: 0; background: inherit; z-index: 2; padding-top: 4px; }
.ro-head{ display:flex; align-items:center; justify-content:space-between; margin-bottom:8px; }
.ro-title{ font-weight:900; }
.ro-actions{ display:flex; gap:8px; }
.ro-body{ overflow:auto; }

/* Tabs small */
.tabs.small { display:flex; gap:6px; }
.tabs.small button{
  flex:1; padding:6px 8px; border:1px solid rgba(255,255,255,.18);
  border-radius:8px; background:rgba(255,255,255,.06); color:inherit; cursor:pointer; font-size:12px; font-weight:700;
}
.tabs.small button.active{ background:rgba(255,255,255,.12) }
.tabs.small.scroll{ overflow-x: auto; -webkit-overflow-scrolling: touch; scrollbar-width: thin; padding-bottom: 2px; }
.tabs.small.scroll button{ flex: 0 0 auto; padding: 6px 10px; }
.tabs.small.scroll::-webkit-scrollbar { height: 6px; }
.tabs.small.scroll::-webkit-scrollbar-thumb { background: rgba(255,255,255,.18); border-radius: 4px; }

/* List rows */
.list{ display:grid; gap:2px; }
.row{
  display:grid; grid-template-columns: auto 1fr auto auto auto auto; gap:6px; align-items:center;
  padding:6px 8px; border-bottom:1px solid rgba(255,255,255,.08); background:transparent; cursor:pointer;
}
.row:last-child{ border-bottom:none; }

/* Top 4 overrides */
.top4-row{ display:flex; gap:8px; align-items:center; margin-bottom:8px; }
.top4-label{ font-size:.9rem; opacity:.9; font-weight:800 }
.top4-select{ padding:6px 8px; border-radius:8px; background:#141a22; color:inherit; border:1px solid rgba(255,255,255,.15) }
.dashboard-root.light .top4-select{ background:#fff; }
.top4-grid{ display:grid; grid-template-columns: repeat(2, minmax(0,1fr)); gap:8px; }
.top4-item{ border:1px solid rgba(255,255,255,.14); background:rgba(255,255,255,.06); border-radius:10px; padding:8px; display:grid; gap:6px; cursor:pointer; }
.top4-item .row1{ display:flex; align-items:center; justify-content:space-between; font-weight:900 }
.top4-item .sym-wrap{ display:flex; align-items:center; gap:8px; }
.top4-item .row2{ display:flex; gap:10px; font-size:.9rem; opacity:.95 }
.top4-item .row3{ display:flex; gap:6px; justify-content:flex-end }

/* Overlay transition */
.drop-enter-active, .drop-leave-active { transition: all .18s ease; }
.drop-enter-from, .drop-leave-to { opacity: 0; transform: translateY(-8px); }

/* 2D scroll utility */
.scroll-2d{ overflow: auto; -webkit-overflow-scrolling: touch; }

/* Colors */
.up{ color:#26a69a } .down{ color:#ef5350 }

/* Pill icon aesthetic */
.dashboard-root { --pill-bg: rgba(255,255,255,.06); --pill-border: rgba(255,255,255,.18); --pill-glow: rgba(255,255,255,.06); --pill-pad-x: 6px; --pill-pad-y: 2px; }
.dashboard-root.light { --pill-bg: rgba(0,0,0,.04); --pill-border: rgba(0,0,0,.12); --pill-glow: rgba(0,0,0,.04); }

.mini-ic, .row-ic, .title-ic {
  display: inline-block; width: 12px; height: 12px; object-fit: contain;
  padding: var(--pill-pad-y) var(--pill-pad-x);
  border-radius: 9999px; background: var(--pill-bg);
  border: 1px solid var(--pill-border); box-shadow: inset 0 0 0 1px var(--pill-glow); opacity: 1 !important;
}
.title-ic { width: 14px; height: 14px; }

.mini-chip{ gap: 4px; padding: 2px 6px; font-size: 11.5px; line-height: 1.1; }
.markets-panel .top4-item .row-ic{ padding: var(--pill-pad-y) var(--pill-pad-x); border-radius: 9999px !important; background: var(--pill-bg); border: 1px solid var(--pill-border); box-shadow: inset 0 0 0 1px var(--pill-glow); }

.row, .watch-row { gap: 6px; padding: 6px 8px; }
</style>
You said:
Please can you review my Dashboard.vue and patch and provide me with a full finalised version - i have been told there are duplicates that need to be removed - Returning the full file back i guess will be to large to return.. what would be the best way to provide me back the final file?


<!-- src/views/Dashboard.vue -->
<template>
  <div
    class="dashboard-root"
    :class="rootClasses"
    @click="closeAllTransientUI"
    data-testid="dashboard-root"
  >
    <!-- ===== FATAL/CHILD ERROR TOAST (non-blocking) ===== -->
    <div v-if="fatalError" class="app-error" role="alert" aria-live="assertive">
      <strong>Something hiccuped:</strong> {{ fatalError }}
      <button class="btn micro ghost" @click="fatalError = ''">Dismiss</button>
    </div>

    <!-- ========== LEFT SIDEBAR ========== -->
    <aside class="sidebar" :class="{ collapsed: sidebarCollapsed }" @click.stop aria-label="Primary">
      <div class="sidebar-top">
        <div class="brand" @click.stop="openHomeInNewTab" role="link" tabindex="0"
             @keydown.enter.prevent="openHomeInNewTab"
             @keydown.space.prevent="openHomeInNewTab">
          <img class="brand-emblem" :src="wickwiseEmblem" alt="WickWise Emblem" />
          <span class="brand-text" v-if="!sidebarCollapsed">WickWise</span>
        </div>
        <button
          class="collapse-btn"
          @click="sidebarCollapsed = !sidebarCollapsed; persistSidebar()"
          :aria-expanded="!sidebarCollapsed"
          :aria-label="sidebarCollapsed ? 'Expand sidebar' : 'Collapse sidebar'"
          data-testid="btn-collapse-sidebar"
        >
          {{ sidebarCollapsed ? '›' : '‹' }}
        </button>
      </div>

      <div v-if="!sidebarCollapsed" class="sidebar-scroll">
        <div class="side-tabs" role="navigation" aria-label="Sidebar">
          <!-- Chart type (dropdown in tab) -->
          <div class="tab-with-menu">
            <button
              :class="{ active: chartMenuOpen }"
              class="side-tab-btn"
              @click.stop="toggleChartTypeMenu"
              aria-haspopup="listbox"
              :aria-expanded="chartMenuOpen ? 'true' : 'false'"
              aria-controls="chart-type-menu"
              title="Select chart type"
              data-testid="btn-chart-type"
            >
              📊 <span>Chart type</span> <span class="caret">▾</span>
            </button>
            <div
              v-if="chartMenuOpen"
              id="chart-type-menu"
              class="tab-menu"
              role="listbox"
              @click.stop
            >
              <button
                class="tab-menu-item"
                role="option"
                :aria-selected="ui.chartMode==='candlestick'"
                :class="{ sel: ui.chartMode==='candlestick' }"
                @click="setChartMode('candlestick')"
                data-testid="opt-chart-candles"
              >
                Candlestick
              </button>
              <button
                class="tab-menu-item"
                role="option"
                :aria-selected="ui.chartMode==='line'"
                :class="{ sel: ui.chartMode==='line' }"
                @click="setChartMode('line')"
                data-testid="opt-chart-line"
              >
                Line
              </button>
            </div>
          </div>

          <!-- Markets now opens a DROP PANEL from the header -->
          <button :class="{ active: leftTab === 'markets' }" @click="openMarketsPanel" title="Markets" data-testid="btn-markets"> 🧭 <span>Markets</span> </button>

          <!-- Zeeno button: wakes Zeeno, modifier = new tab -->
          <button :class="{ active: true }" @click="openZeeno" title="Zeeno (Auto-Analyse)" data-testid="btn-zeeno">🧠 <span>Zeeno</span></button>

          <button :class="{ active: false }" @click="instantToggleTheme" title="Toggle theme" data-testid="btn-theme">🌓 <span>Theme</span></button>

          <button :class="{ active: leftTab === 'pairs' }" @click="openOverlay('pairs')" title="Pairs" data-testid="btn-pairs">🔗 <span>Pairs</span></button>

          <!-- Settings (toggles drop-down panel) & Logout -->
          <button :class="{ active: settingsOpen }" @click="toggleSettings" title="Settings" data-testid="btn-settings">⚙️ <span>Settings</span></button>
          <button :class="{ active: false }" @click="logout" title="Logout" data-testid="btn-logout">🚪 <span>Logout</span></button>
        </div>
      </div>
    </aside>

    <!-- ========== MAIN ========== -->
    <main class="main" @click.stop>
      <!-- Optional service notice / empty state -->
      <div v-if="serviceNotice" class="service-notice" role="alert" aria-live="polite" data-testid="service-notice">
        {{ serviceNotice }}
      </div>

      <!-- ===== POWERED BY / PROVIDERS BAR (moved ABOVE top movers) ===== -->
      <div class="powerbar" role="contentinfo" aria-label="Data providers">
        <span class="pb-title">Powered by</span>
        <ul class="pb-list" v-if="providerChips.length">
          <li
            v-for="p in providerChips"
            :key="p.id"
            class="pb-chip"
            :class="{ off: !p.ok }"
            :title="p.title"
          >
            <img class="pb-ic" :src="p.icon" :alt="p.name" @error="onProviderIconError($event, p)" />
            <span class="pb-name">{{ p.name }}</span>
            <span class="pb-dot" :class="p.ok ? 'ok' : 'warn'"></span>
          </li>
        </ul>
        <span class="pb-pill" :class="{ warn: !serverLiveData }">
          {{ serverLiveData ? 'Live data' : 'Fallback mode' }}
        </span>
      </div>

      <!-- ===== SETTINGS DROP PANEL (restored) ===== -->
      <transition name="settings-drop">
        <section
          v-if="settingsOpen"
          ref="settingsEl"
          class="settings-panel"
          role="dialog"
          aria-modal="true"
          aria-label="Settings"
          tabindex="0"
          @click.stop
        >
          <div class="sp-head">
            <h3 class="sp-title">Settings</h3>
            <div class="sp-actions">
              <button class="btn tiny" @click="toggleSettings">Close</button>
            </div>
          </div>
          <div class="sp-body">
            <div style="display:grid; gap:8px;">
              <div><strong>Theme:</strong> <button class="btn tiny" @click="instantToggleTheme">Toggle ({{ theme }})</button></div>
              <div>
                <strong>Timeframe:</strong>
                <select v-model="ui.tf" class="cc-select" data-testid="settings-tf">
                  <option>1m</option><option>5m</option><option>15m</option>
                  <option>1h</option><option>4h</option><option>1d</option>
                </select>
              </div>
              <div><strong>Subscription:</strong> {{ devPlanLabel }}</div>
              <div><strong>User ID:</strong> <code>{{ userId }}</code></div>
            </div>
          </div>
        </section>
      </transition>

      <!-- HEADER (Top Movers chips) -->
      <div class="topbar" @click.stop>
        <div class="mini-movers" aria-live="polite">
          <span class="mini-title">Top Movers:</span>
          <template v-if="headerMovers.length">
            <button
              v-for="m in headerMovers"
              :key="m.asset + ':' + m.symbol"
              class="mini-chip"
              type="button"
              @click="onTopMoverClick($event, m)"
              @keydown.enter.prevent="onTopMoverClick($event, m)"
              @keydown.space.prevent="onTopMoverClick($event, m)"
              @dblclick="openFullChartNewTab({ key:m.id || m.symbol, class:m.asset })"
              :title="${prettyMarket(m.asset)} · ${m.symbol} · ${formatPrice(m.price)} · ${signedPct(m.change)} · Alt-click to pin"
              data-testid="mini-mover"
            >
              <img class="mini-ic" :src="iconFor(m.asset, m.symbol)" :alt="m.symbol" @error="onIconError($event, m.asset)" />
              <span class="mini-market">{{ prettyMarket(m.asset) }}</span>
              <span class="mini-symbol">{{ m.symbol }}</span>
              <span class="mini-chg" :class="m.change >=0 ? 'up':'down'">{{ signedPct(m.change) }}</span>
            </button>
          </template>
          <span v-else class="mini-hint">Loading movers…</span>
          <button class="btn micro ghost" style="margin-left:6px" @click="refreshTop" title="Reload movers" data-testid="btn-reload-movers">Reload</button>
        </div>
      </div>

      <!-- ====== MARKETS DROP PANEL (from header) ====== -->
      <transition name="markets-drop">
        <section
          v-if="marketPanel.open"
          class="markets-panel"
          role="dialog"
          aria-modal="true"
          aria-label="Markets"
          tabindex="0"
          ref="marketEl"
          @click.stop
          data-testid="panel-markets"
          @keydown.esc.stop.prevent="closeMarketsPanel"
        >
          <div class="mp-head">
            <h3 class="mp-title">Markets</h3>
            <div class="mp-actions">
              <div class="tabs small" style="margin-right:8px;">
                <button v-if="tvEnabled" :class="{ active: marketsView==='overview' }" @click="marketsView='overview'" data-testid="tab-overview">Overview</button>
                <button :class="{ active: marketsView==='list' || !tvEnabled }" @click="marketsView='list'" data-testid="tab-list">List</button>
                <!-- NEW: Top 4 tab renders INSIDE the Markets panel -->
                <button :class="{ active: marketsView==='top4' }" @click="openTop4InPanel" data-testid="tab-top4">Top 4</button>
              </div>
              <button class="btn tiny" @click="refreshTop">Reload</button>
              <button class="btn tiny" @click="closeMarketsPanel">Close</button>
            </div>
          </div>

          <div class="mp-body">
            <div v-if="tvEnabled && marketsView==='overview'" class="mp-overview disabled">
              <div class="empty small" style="padding:10px;">Market Overview embeds are disabled.</div>
            </div>

            <div v-else-if="marketsView==='list' || !tvEnabled" class="mp-list scroll-2d">
              <div class="tabs small scroll" v-hscroll style="margin-bottom:8px;">
                <button v-for="t in drawerTabs" :key="t.key" :class="{ active: marketTab === t.key }" @click="marketTab = t.key">{{ t.label }}</button>
              </div>

              <div class="mp-list-inner" :aria-busy="!visibleDrawerItems.length ? 'true' : undefined">
                <div
                  v-for="it in visibleDrawerItems"
                  :key="it.key"
                  class="row"
                  @click="quickSelectFromChip(it)"
                  @dblclick="openFullChartNewTab(it)"
                  :title="${it.label} · ${formatPrice(it.price)} · ${signedPct(it.change)}"
                  data-testid="row-market-item"
                >
                  <img class="row-ic" :src="it.icon || iconFor(it.asset, it.symbol || it.key)" :alt="it.label" @error="onIconError($event, it.asset)" />
                  <span class="row-name">{{ it.label }}</span>
                  <span class="row-price">{{ formatPrice(it.price) }}</span>
                  <span class="row-chg" :class="(it.change ?? 0) >= 0 ? 'up' : 'down'">{{ signedPct(it.change) }}</span>
                  <button
                    class="icon-btn pin"
                    :class="{ active: inWatchlist(it, it.asset) }"
                    @click.stop="toggleWatchlist(it, it.asset)"
                    :title="inWatchlist(it, it.asset) ? 'Unpin' : 'Pin'"
                    aria-label="Pin"
                    :aria-pressed="inWatchlist(it, it.asset) ? 'true' : 'false'"
                    data-testid="btn-pin-market-item"
                  >
                    <svg viewBox="0 0 24 24" class="ic"><path d="M14 2l1 5 3 3-5 1-6 8-1-1 8-6 1-5z"/></svg>
                  </button>
                </div>
                <div class="empty" v-if="!visibleDrawerItems.length">No data yet. Check API keys on the server.</div>
              </div>
            </div>

            <!-- NEW: Top-4 rendered INSIDE Markets panel -->
            <div v-else-if="marketsView==='top4'" class="mp-list scroll-2d" data-testid="panel-top4">
              <div class="top4-row">
                <label class="top4-label">Market</label>
                <select v-model="top4Market" class="top4-select" @change="loadTop4ForMarket" data-testid="select-top4-market">
                  <option v-for="opt in marketOptions" :key="opt.key" :value="opt.key">{{ opt.label }}</option>
                </select>
                <button class="btn tiny" @click="loadTop4ForMarket">Reload</button>
              </div>

              <div class="top4-grid">
                <div
                  v-for="it in top4Visible"
                  :key="it.key"
                  class="top4-item"
                  @click="quickSelectFromChip(it)"
                  :title="${it.label} · C ${formatPrice(it.close)} · H ${formatPrice(it.high)} · L ${formatPrice(it.low)}"
                >
                  <div class="row1">
                    <div class="sym-wrap">
                      <img class="row-ic" :src="iconFor(it.asset, it.symbol)" :alt="it.label" @error="onIconError($event, it.asset)" />
                      <span class="sym">{{ it.label }}</span>
                    </div>
                    <span class="chg" :class="(it.change ?? 0) >= 0 ? 'up' : 'down'">{{ signedPct(it.change) }}</span>
                  </div>
                  <div class="row2">
                    <span class="px">C {{ formatPrice(it.close ?? it.price) }}</span>
                    <span class="px">H {{ formatPrice(it.high) }}</span>
                    <span class="px">L {{ formatPrice(it.low) }}</span>
                  </div>
                  <div class="row3">
                    <button class="btn micro" @click.stop="openFullChartNewTab({ key: it.id || it.symbol, class: it.asset })">Full chart</button>
                    <button
                      class="icon-btn pin"
                      :class="{ active: inWatchlist(it, it.asset) }"
                      @click.stop="pinTop4(it)"
                      :title="inWatchlist(it, it.asset) ? 'Unpin' : 'Pin'"
                      aria-label="Pin"
                      :aria-pressed="inWatchlist(it, it.asset) ? 'true' : 'false'"
                    >
                      <svg viewBox="0 0 24 24" class="ic"><path d="M14 2l1 5 3 3-5 1-6 8-1-1 8-6 1-5z"/></svg>
                    </button>
                  </div>
                </div>
              </div>
            </div>
            <!-- /Top-4 in Markets panel -->
          </div>
        </section>
      </transition>

      <!-- KPI + AI Overview + Actions -->
      <div class="kpi-row">
        <div class="kpi-card">
          <div class="kpi-title">Selected</div>
          <div class="kpi-selected">
            <div class="kpi-value">
              <img
                class="title-ic"
                :src="iconFor(activeClass, activeClass==='crypto' ? (activeName || activeSymbol) : activeSymbol)"
                :alt="selectedLabel"
                @error="onIconError($event, activeClass)"
              />
              <span>{{ selectedLabel }}</span>
            </div>
            <div class="kpi-sub" :class="(changeForSelected ?? 0) >= 0 ? 'up' : 'down'">
              {{ signedPct(changeForSelected) }}
            </div>
          </div>
        </div>

        <div class="kpi-card highlight">
          <div class="kpi-title">AI Overview — {{ selectedLabel }}</div>
          <div class="ai-overview" v-if="aiOverview.ready">
            <div class="ao-headline">{{ aiOverview.title }}</div>
            <div class="ao-tags">
              <span class="tag" :class="aiOverview.momentumClass">{{ aiOverview.momentum }}</span>
              <span class="tag" :class="aiOverview.regimeClass">{{ aiOverview.regime }}</span>
              <span class="tag">{{ aiOverview.volatility }}</span>
              <span class="tag ghost">live</span>
            </div>
            <p class="ao-summary">{{ aiOverview.summary }}</p>
            <ul class="ao-bullets">
              <li v-for="(b, i) in aiOverview.bullets" :key="i">{{ b }}</li>
            </ul>
            <div class="ao-foot"><span class="muted">Zeeno read:</span> {{ aiOverview.zeenoTake }}</div>
          </div>
          <div class="ai-overview empty" v-else>
            <div class="ao-headline">Gathering data…</div>
            <p class="ao-summary">Waiting for enough candles to compile a session recap and live trend read.</p>
          </div>
        </div>

        <div class="kpi-card">
          <div class="kpi-title">Actions</div>
          <div class="kpi-actions spaced" role="group" aria-label="Chart actions">
            <button class="btn tiny" @click="refreshAll" data-testid="btn-refresh-all">Refresh</button>
            <button class="btn tiny" @click="openFullChartFromCurrentNewTab" data-testid="btn-open-full-chart">Full chart</button>
          </div>
        </div>
      </div>

      <!-- ===== CHART ===== -->
      <section class="section chart-section" aria-label="Chart section">
        <!-- Chart toolbar -->
        <div class="chart-controls" role="toolbar" aria-label="Chart navigation">
          <div class="toolbar">
            <button class="btn micro" @click="panLeft" data-testid="btn-pan-left" aria-label="Pan left">←</button>
            <button class="btn micro" @click="zoomOut" data-testid="btn-zoom-out" aria-label="Zoom out">−</button>
            <button class="btn micro" @click="resetView" data-testid="btn-reset-view" aria-label="Reset view">Reset</button>
            <button class="btn micro" @click="zoomIn" data-testid="btn-zoom-in" aria-label="Zoom in">+</button>
            <button class="btn micro" @click="panRight" data-testid="btn-pan-right" aria-label="Pan right">→</button>
          </div>

          <div class="spacer"></div>

          <label for="tf-select">TF</label>
          <select v-model="ui.tf" class="cc-select" id="tf-select" data-testid="select-tf" aria-label="Timeframe">
            <option>1m</option><option>5m</option><option>15m</option>
            <option>1h</option><option>4h</option><option>1d</option>
          </select>

          <label class="cc-check"><input type="checkbox" v-model="ui.logScale" /> Log</label>
        </div>

        <!-- Chart block with overlay -->
        <div class="chart-block" :aria-busy="loading.price ? 'true' : 'false'" data-testid="chart-block">
          <div class="chart-overlay">
            <button class="btn micro ghost" title="Refresh chart" @click="refreshChart" data-testid="btn-refresh-chart">Refresh</button>
            <button class="btn micro ghost" title="Open full chart" @click="openFullChartFromCurrentNewTab">Full chart</button>
          </div>

          <WidgetShell @retry="reloadChart++" class="chart-shell">
            <!-- Chart is now rendered by ChartCard (emitter), NOT MarketChart -->
            <ChartCard
              :key="chartKey"
              :symbol="resolveApiTarget(activeClass, activeSymbol).symbol"
              :asset="resolveApiTarget(activeClass, activeSymbol).asset"
              :timeframe="ui.tf"
              :theme="theme"
              :height="'100%'"
              :candles="visibleCandles"
              :loading="loading.price"
              :error="errors.price"
              :chartType="marketChartType"
              :maxPoints="1200"
              :logScale="ui.logScale"
              :barSpacing="ui.barSpacing"
              :showVolume="ui.show.volume"
              @open="openFullChartFromCurrentNewTab"
              @crosshair="onCrosshair"
              @hover="onCrosshair"
              @providers="ingestProviders"
              @attribution="addProvidersFromText"
            />
          </WidgetShell>

          <!-- Accessible inline error, if any -->
          <div v-if="errors.price" class="chart-error" role="alert" aria-live="assertive" data-testid="chart-error">
            {{ errors.price }}
          </div>
        </div>

        <!-- OHLC panel -->
        <div class="ohlc-card" v-if="ohlc.ready" aria-live="polite" data-testid="ohlc-card">
          <div class="ohlc-row">
            <div class="ohlc-item"><span>Open</span><strong>{{ formatPrice(ohlc.open) }}</strong></div>
            <div class="ohlc-item"><span>High</span><strong>{{ formatPrice(ohlc.high) }}</strong></div>
            <div class="ohlc-item"><span>Low</span><strong>{{ formatPrice(ohlc.low) }}</strong></div>
            <div class="ohlc-item">
              <span>Close</span>
              <strong :class="ohlc.changePct>=0 ? 'up':'down'">
                {{ formatPrice(ohlc.close) }} ({{ signedPct(ohlc.changePct) }})
              </strong>
            </div>
            <div class="ohlc-item" v-if="Number.isFinite(ohlc.volume)"><span>Volume</span><strong>{{ formatVol(ohlc.volume) }}</strong></div>
            <div class="ohlc-item" v-if="ohlc.time"><span>Time</span><strong>{{ prettyTime(ohlc.time) }}</strong></div>
          </div>
        </div>
      </section>
    </main>

    <!-- ========== RIGHT RAIL ========== -->
    <aside class="right-rail" @click.stop aria-label="Secondary">
      <!-- Watchlist (moved ABOVE news) -->
      <div class="watch-card">
        <div class="watch-head">
          <h3 class="section-title small">Watchlist</h3>
          <div class="watch-actions">
            <button class="btn tiny" v-if="watchlist.length" @click="clearWatchlist" aria-label="Clear watchlist" data-testid="btn-clear-watch">Clear</button>
          </div>
        </div>

        <div v-if="!watchlist.length" class="empty small">No pins yet. Use Markets / Pairs / Top 4.</div>

        <ul v-else class="watch compact" data-testid="watchlist">
          <li
            v-for="w in pagedWatch"
            :key="w.key"
            class="watch-row compact"
            :class="{ active: isSelectedWatch(w) }"
            @click="quickSelect(w)"
            @dblclick="openFullChartFromWatchNewTab(w)"
            :title="${w.label} · ${formatPrice(latestPriceForWatch(w))} · ${signedPct(latestChangeForWatch(w))}"
          >
            <img class="row-ic" :src="iconFor(w.type, w.meta?.symbol || w.meta?.id || w.label)" :alt="w.label" @error="onIconError($event, w.type)" />
            <span class="row-name">{{ w.label }}</span>
            <span class="row-price">{{ formatPrice(latestPriceForWatch(w)) }}</span>
            <span class="row-chg" :class="(latestChangeForWatch(w) ?? 0) >= 0 ? 'up' : 'down'">
              {{ signedPct(latestChangeForWatch(w)) }}
            </span>
            <button
              class="icon-btn pin"
              :class="{ active: inWatchlist(w.meta || w, w.type) }"
              @click.stop="toggleWatchlist(w.meta || w, w.type)"
              :title="inWatchlist(w.meta || w, w.type) ? 'Unpin' : 'Pin'"
              aria-label="Pin"
              :aria-pressed="inWatchlist(w.meta || w, w.type) ? 'true' : 'false'"
            >
              <svg viewBox="0 0 24 24" class="ic"><path d="M14 2l1 5 3 3-5 1-6 8-1-1 8-6 1-5z"/></svg>
            </button>
            <button class="wl-x" @click.stop="removeFromWatchlist(w)" title="Remove" aria-label="Remove from watchlist">✕</button>
          </li>
        </ul>

        <div class="pager compact" v-if="watchlistTotalPages > 1">
          <button class="pg-btn" :disabled="watchlistPage === 1" @click="watchlistPage--" aria-label="Previous page">‹</button>
          <span class="pg-ind">{{ watchlistPage }} / {{ watchlistTotalPages }}</span>
          <button class="pg-btn" :disabled="watchlistPage === watchlistTotalPages" @click="watchlistPage++" aria-label="Next page">›</button>
        </div>
      </div>

      <!-- Live/Global News with COUNT TABS + RELOAD -->
      <div class="news-card rail">
        <div class="rail-head">
          <h3 class="section-title small">News</h3>

          <!-- PATCH A: Count tabs for both news feeds -->
          <div class="tabs small count-tabs">
            <button
              :class="{ active: newsTab==='relevant' }"
              @click="newsTab='relevant'"
              data-testid="tab-news-relevant"
            >
              Relevant
              <span class="badge">{{ relevantCount }}</span>
            </button>
            <button
              :class="{ active: newsTab==='global' }"
              @click="newsTab='global'"
              data-testid="tab-news-global"
            >
              Global
              <span class="badge">{{ globalCount }}</span>
            </button>
          </div>

          <!-- PATCH B: One-click reload for BOTH lists -->
          <div class="watch-actions">
            <button class="btn tiny" @click="reloadNews" title="Reload both news lists" data-testid="btn-reload-news">Reload</button>
          </div>
        </div>

        <!-- ===== Relevant (selection-specific) ===== -->
        <template v-if="newsTab==='relevant'">
          <div class="rail-subtitle">Live — {{ selectedLabel }}</div>

          <div v-if="newsError" class="news-error" role="alert" aria-live="assertive" data-testid="news-error">{{ newsError }}</div>

          <!-- Skeleton while loading -->
          <ul v-if="newsLoading" class="news-list rail">
            <li class="news-item rail" v-for="i in 3" :key="'nsk'+i">
              <div class="skeleton-row"></div>
              <div class="meta-row">
                <span class="chip ghost skeleton-chip"></span>
                <span class="chip ghost skeleton-chip"></span>
              </div>
            </li>
          </ul>

          <div v-else class="news-live" aria-live="polite">
            <template v-if="!newsError">
              <ul v-if="filteredNewsItems.length" class="news-list rail" data-testid="news-list">
                <li v-for="n in visibleNews" :key="n.id" class="news-item rail">
                  <a class="news-link clamp-2" :href="n.url" target="_blank" rel="noopener">{{ n.headline }}</a>
                  <div class="meta-row">
                    <span class="chip">{{ n.source }}</span>
                    <span class="dot">·</span>
                    <span class="chip ghost">{{ prettyTime(n.datetime) }}</span>
                  </div>
                </li>
              </ul>
              <div v-else-if="newsHintLink" class="news-hint" data-testid="news-hint">
                No direct headlines. Try <a :href="newsHintLink" target="_blank" rel="noopener">Google News</a>.
              </div>
              <div v-else class="news-empty">No news available.</div>

              <div v-if="filteredNewsItems.length > newsLimit" class="news-more">
                <button class="btn tiny" @click="newsLimit += 10">Show more</button>
              </div>
            </template>
          </div>

          <div v-if="newsAttribution" class="news-attr">{{ newsAttribution }}</div>
        </template>

        <!-- ===== Global (asset-wide) ===== -->
        <template v-else>
          <div class="rail-subtitle">Global — {{ prettyMarket(normalizeAssetParam(activeClass)) }}</div>

          <div v-if="globalNewsError" class="news-error" role="alert" aria-live="assertive" data-testid="global-news-error">{{ globalNewsError }}</div>

          <!-- Skeleton while loading -->
          <ul v-if="globalNewsLoading" class="news-list rail">
            <li class="news-item rail" v-for="i in 3" :key="'gnsk'+i">
              <div class="skeleton-row"></div>
              <div class="meta-row">
                <span class="chip ghost skeleton-chip"></span>
                <span class="chip ghost skeleton-chip"></span>
              </div>
            </li>
          </ul>

          <div v-else class="news-live" aria-live="polite">
            <template v-if="!globalNewsError">
              <ul v-if="visibleGlobalNews.length" class="news-list rail" data-testid="news-list-global">
                <li v-for="n in visibleGlobalNews" :key="n.id" class="news-item rail">
                  <a class="news-link clamp-2" :href="n.url" target="_blank" rel="noopener">{{ n.headline }}</a>
                  <div class="meta-row">
                    <span class="chip">{{ n.source }}</span>
                    <span class="dot">·</span>
                    <span class="chip ghost">{{ prettyTime(n.datetime) }}</span>
                  </div>
                </li>
              </ul>
              <div v-else-if="globalNewsHintLink" class="news-hint" data-testid="global-news-hint">
                No global headlines. Try <a :href="globalNewsHintLink" target="_blank" rel="noopener">Google News</a>.
              </div>
              <div v-else class="news-empty">No news available.</div>

              <div v-if="globalNewsItems.length > globalNewsLimit" class="news-more">
                <button class="btn tiny" @click="globalNewsLimit += 10">Show more</button>
              </div>
            </template>
          </div>

          <div v-if="globalNewsAttribution" class="news-attr">{{ globalNewsAttribution }}</div>
        </template>
      </div>

      <!-- Right Overlay (PAIRS only) -->
      <transition name="drop">
        <div
          v-if="rightOverlay.open"
          class="right-overlay"
          :class="{ floating: isCompact }"
          role="dialog"
          aria-modal="true"
          :aria-labelledby="rightOverlay.kind ? (rightOverlay.kind + '-title') : undefined"
          @keydown.esc="closeOverlay"
          tabindex="0"
          ref="overlayEl"
          data-testid="right-overlay"
        >
          <div class="ro-head">
            <div class="ro-title" :id="rightOverlay.kind ? (rightOverlay.kind + '-title') : null">
              <template v-if="rightOverlay.kind==='pairs'">Pair &amp; Selectors</template>
            </div>
            <div class="ro-actions">
              <button class="btn tiny" @click="closeOverlay">Close</button>
            </div>
          </div>

          <div v-if="rightOverlay.kind==='pairs'" class="ro-body">
            <div class="tabs small" style="margin-bottom:8px;">
              <button :class="{ active: activeTab === 'forex' }" @click="setTab('forex')" data-testid="tab-forex">Forex</button>
              <button :class="{ active: activeTab === 'crypto' }" @click="setTab('crypto')" data-testid="tab-crypto">Crypto</button>
            </div>

            <div class="list">
              <template v-if="activeTab === 'forex'">
                <div
                  v-for="p in fxPairs"
                  :key="p.symbol"
                  class="row"
                  @click="selectPair(p, 'fx')"
                  @dblclick="openFullChartNewTab({ key: p.symbol, label: p.symbol, class: 'fx' })"
                >
                  <img class="row-ic" :src="p.icon" :alt="p.symbol" @error="onIconError($event, 'fx')" />
                  <span class="row-name">{{ p.symbol }}</span>
                  <button
                    class="icon-btn pin"
                    :class="{ active: inWatchlist(p, 'fx') }"
                    @click.stop="toggleWatchlist(p, 'fx')"
                    :title="inWatchlist(p, 'fx') ? 'Unpin' : 'Pin'"
                    aria-label="Pin"
                    :aria-pressed="inWatchlist(p, 'fx') ? 'true' : 'false'"
                  >
                    <svg viewBox="0 0 24 24" class="ic"><path d="M14 2l1 5 3 3-5 1-6 8-1-1 8-6 1-5z"/></svg>
                  </button>
                </div>
              </template>

              <template v-else>
                <div
                  v-for="c in cryptoCoins"
                  :key="c.id"
                  class="row"
                  @click="selectPair({ id: c.id, symbol: c.symbol, name: c.name }, 'crypto')"
                  @dblclick="openFullChartNewTab({ key: c.id, label: c.symbol, class: 'crypto' })"
                >
                  <img class="row-ic" :src="c.icon" :alt="c.symbol" @error="onIconError($event, 'crypto')" />
                  <span class="row-name">{{ c.symbol }}</span>
                  <button
                    class="icon-btn pin"
                    :class="{ active: inWatchlist(c, 'crypto') }"
                    @click.stop="toggleWatchlist(c, 'crypto')"
                    :title="inWatchlist(c, 'crypto') ? 'Unpin' : 'Pin'"
                    aria-label="Pin"
                    :aria-pressed="inWatchlist(c, 'crypto') ? 'true' : 'false'"
                  >
                    <svg viewBox="0 0 24 24" class="ic"><path d="M14 2l1 5 3 3-5 1-6 8-1-1 8-6 1-5z"/></svg>
                  </button>
                </div>
              </template>
            </div>
          </div>
        </div>
      </transition>
    </aside>

    <!-- Mount Zeeno once; it manages its own floating window -->
    <KeepAlive>
      <Zeeno v-bind="zeenoProps" />
    </KeepAlive>
  </div>
</template>
      <!-- KPI + AI Overview + Actions -->
      <div class="kpi-row">
        <div class="kpi-card">
          <div class="kpi-title">Selected</div>
          <div class="kpi-selected">
            <div class="kpi-value">
              <img
                class="title-ic"
                :src="iconFor(activeClass, activeClass==='crypto' ? (activeName || activeSymbol) : activeSymbol)"
                :alt="selectedLabel"
                @error="onIconError($event, activeClass)"
              />
              <span>{{ selectedLabel }}</span>
            </div>
            <div class="kpi-sub" :class="(changeForSelected ?? 0) >= 0 ? 'up' : 'down'">
              {{ signedPct(changeForSelected) }}
            </div>
          </div>
        </div>

        <div class="kpi-card highlight">
          <div class="kpi-title">AI Overview — {{ selectedLabel }}</div>
          <div class="ai-overview" v-if="aiOverview.ready">
            <div class="ao-headline">{{ aiOverview.title }}</div>
            <div class="ao-tags">
              <span class="tag" :class="aiOverview.momentumClass">{{ aiOverview.momentum }}</span>
              <span class="tag" :class="aiOverview.regimeClass">{{ aiOverview.regime }}</span>
              <span class="tag">{{ aiOverview.volatility }}</span>
              <span class="tag ghost">live</span>
            </div>
            <p class="ao-summary">{{ aiOverview.summary }}</p>
            <ul class="ao-bullets">
              <li v-for="(b, i) in aiOverview.bullets" :key="i">{{ b }}</li>
            </ul>
            <div class="ao-foot"><span class="muted">Zeeno read:</span> {{ aiOverview.zeenoTake }}</div>
          </div>
          <div class="ai-overview empty" v-else>
            <div class="ao-headline">Gathering data…</div>
            <p class="ao-summary">Waiting for enough candles to compile a session recap and live trend read.</p>
          </div>
        </div>

        <div class="kpi-card">
          <div class="kpi-title">Actions</div>
          <div class="kpi-actions spaced" role="group" aria-label="Chart actions">
            <button class="btn tiny" @click="refreshAll" data-testid="btn-refresh-all">Refresh</button>
            <button class="btn tiny" @click="openFullChartFromCurrentNewTab" data-testid="btn-open-full-chart">Full chart</button>
          </div>
        </div>
      </div>

      <!-- ===== CHART ===== -->
      <section class="section chart-section" aria-label="Chart section">
        <!-- Chart toolbar -->
        <div class="chart-controls" role="toolbar" aria-label="Chart navigation">
          <div class="toolbar">
            <button class="btn micro" @click="panLeft" data-testid="btn-pan-left" aria-label="Pan left">←</button>
            <button class="btn micro" @click="zoomOut" data-testid="btn-zoom-out" aria-label="Zoom out">−</button>
            <button class="btn micro" @click="resetView" data-testid="btn-reset-view" aria-label="Reset view">Reset</button>
            <button class="btn micro" @click="zoomIn" data-testid="btn-zoom-in" aria-label="Zoom in">+</button>
            <button class="btn micro" @click="panRight" data-testid="btn-pan-right" aria-label="Pan right">→</button>
          </div>

          <div class="spacer"></div>

          <label for="tf-select">TF</label>
          <select v-model="ui.tf" class="cc-select" id="tf-select" data-testid="select-tf" aria-label="Timeframe">
            <option>1m</option><option>5m</option><option>15m</option>
            <option>1h</option><option>4h</option><option>1d</option>
          </select>

          <label class="cc-check"><input type="checkbox" v-model="ui.logScale" /> Log</label>
        </div>

        <!-- Chart block with overlay -->
        <div class="chart-block" :aria-busy="loading.price ? 'true' : 'false'" data-testid="chart-block">
          <div class="chart-overlay">
            <button class="btn micro ghost" title="Refresh chart" @click="refreshChart" data-testid="btn-refresh-chart">Refresh</button>
            <button class="btn micro ghost" title="Open full chart" @click="openFullChartFromCurrentNewTab">Full chart</button>
          </div>

          <WidgetShell @retry="reloadChart++" class="chart-shell">
            <ChartCard
              :key="chartKey"
              :symbol="resolveApiTarget(activeClass, activeSymbol).symbol"
              :asset="resolveApiTarget(activeClass, activeSymbol).asset"
              :timeframe="ui.tf"
              :theme="theme"
              :height="'100%'"
              :candles="visibleCandles"
              :loading="loading.price"
              :error="errors.price"
              :chartType="marketChartType"
              :maxPoints="1200"
              :logScale="ui.logScale"
              :barSpacing="ui.barSpacing"
              :showVolume="ui.show.volume"
              @open="openFullChartFromCurrentNewTab"
              @crosshair="onCrosshair"
              @hover="onCrosshair"
              @providers="ingestProviders"
              @attribution="addProvidersFromText"
            />
          </WidgetShell>

          <!-- Accessible inline error, if any -->
          <div v-if="errors.price" class="chart-error" role="alert" aria-live="assertive" data-testid="chart-error">
            {{ errors.price }}
          </div>
        </div>

        <!-- OHLC panel -->
        <div class="ohlc-card" v-if="ohlc.ready" aria-live="polite" data-testid="ohlc-card">
          <div class="ohlc-row">
            <div class="ohlc-item"><span>Open</span><strong>{{ formatPrice(ohlc.open) }}</strong></div>
            <div class="ohlc-item"><span>High</span><strong>{{ formatPrice(ohlc.high) }}</strong></div>
            <div class="ohlc-item"><span>Low</span><strong>{{ formatPrice(ohlc.low) }}</strong></div>
            <div class="ohlc-item">
              <span>Close</span>
              <strong :class="ohlc.changePct>=0 ? 'up':'down'">
                {{ formatPrice(ohlc.close) }} ({{ signedPct(ohlc.changePct) }})
              </strong>
            </div>
            <div class="ohlc-item" v-if="Number.isFinite(ohlc.volume)"><span>Volume</span><strong>{{ formatVol(ohlc.volume) }}</strong></div>
            <div class="ohlc-item" v-if="ohlc.time"><span>Time</span><strong>{{ prettyTime(ohlc.time) }}</strong></div>
          </div>
        </div>
      </section>
    </main>

    <!-- ========== RIGHT RAIL ========== -->
    <aside class="right-rail" @click.stop aria-label="Secondary">
      <!-- Watchlist (moved ABOVE news) -->
      <div class="watch-card">
        <div class="watch-head">
          <h3 class="section-title small">Watchlist</h3>
          <div class="watch-actions">
            <button class="btn tiny" v-if="watchlist.length" @click="clearWatchlist" aria-label="Clear watchlist" data-testid="btn-clear-watch">Clear</button>
          </div>
        </div>

        <div v-if="!watchlist.length" class="empty small">No pins yet. Use Markets / Pairs / Top 4.</div>

        <ul v-else class="watch compact" data-testid="watchlist">
          <li
            v-for="w in pagedWatch"
            :key="w.key"
            class="watch-row compact"
            :class="{ active: isSelectedWatch(w) }"
            @click="quickSelect(w)"
            @dblclick="openFullChartFromWatchNewTab(w)"
            :title="${w.label} · ${formatPrice(latestPriceForWatch(w))} · ${signedPct(latestChangeForWatch(w))}"
          >
            <img class="row-ic" :src="iconFor(w.type, w.meta?.symbol || w.meta?.id || w.label)" :alt="w.label" @error="onIconError($event, w.type)" />
            <span class="row-name">{{ w.label }}</span>
            <span class="row-price">{{ formatPrice(latestPriceForWatch(w)) }}</span>
            <span class="row-chg" :class="(latestChangeForWatch(w) ?? 0) >= 0 ? 'up' : 'down'">
              {{ signedPct(latestChangeForWatch(w)) }}
            </span>
            <button
              class="icon-btn pin"
              :class="{ active: inWatchlist(w.meta || w, w.type) }"
              @click.stop="toggleWatchlist(w.meta || w, w.type)"
              :title="inWatchlist(w.meta || w, w.type) ? 'Unpin' : 'Pin'"
              aria-label="Pin"
              :aria-pressed="inWatchlist(w.meta || w, w.type) ? 'true' : 'false'"
            >
              <svg viewBox="0 0 24 24" class="ic"><path d="M14 2l1 5 3 3-5 1-6 8-1-1 8-6 1-5z"/></svg>
            </button>
            <button class="wl-x" @click.stop="removeFromWatchlist(w)" title="Remove" aria-label="Remove from watchlist">✕</button>
          </li>
        </ul>

        <div class="pager compact" v-if="watchlistTotalPages > 1">
          <button class="pg-btn" :disabled="watchlistPage === 1" @click="watchlistPage--" aria-label="Previous page">‹</button>
          <span class="pg-ind">{{ watchlistPage }} / {{ watchlistTotalPages }}</span>
          <button class="pg-btn" :disabled="watchlistPage === watchlistTotalPages" @click="watchlistPage++" aria-label="Next page">›</button>
        </div>
      </div>

      <!-- News (tabs with counts + reload for both) -->
      <div class="news-card rail">
        <div class="rail-head" style="gap:8px;">
          <h3 class="section-title small">News</h3>
          <div class="tabs small" style="margin-left:auto;">
            <button :class="{ active: newsTab==='relevant' }" @click="newsTab='relevant'" data-testid="tab-news-relevant">
              Relevant <span class="chip ghost">{{ filteredNewsItems.length }}</span>
            </button>
            <button :class="{ active: newsTab==='global' }" @click="newsTab='global'; if(!globalNewsItems.length) refreshGlobalNews()" data-testid="tab-news-global">
              Global <span class="chip ghost">{{ globalNewsItems.length }}</span>
            </button>
            <button class="btn tiny" @click="reloadNewsBoth" data-testid="btn-reload-news">Reload</button>
          </div>
        </div>

        <!-- Tab: Relevant (by selection) -->
        <template v-if="newsTab==='relevant'">
          <div v-if="newsError" class="news-error" role="alert" aria-live="assertive" data-testid="news-error">{{ newsError }}</div>

          <!-- Skeleton while loading -->
          <ul v-if="newsLoading" class="news-list rail">
            <li class="news-item rail" v-for="i in 3" :key="'nsk'+i">
              <div class="skeleton-row"></div>
              <div class="meta-row">
                <span class="chip ghost skeleton-chip"></span>
                <span class="chip ghost skeleton-chip"></span>
              </div>
            </li>
          </ul>

          <div v-else class="news-live" aria-live="polite">
            <template v-if="!newsError">
              <ul v-if="filteredNewsItems.length" class="news-list rail" data-testid="news-list">
                <li v-for="n in visibleNews" :key="n.id" class="news-item rail">
                  <a class="news-link clamp-2" :href="n.url" target="_blank" rel="noopener">{{ n.headline }}</a>
                  <div class="meta-row">
                    <span class="chip">{{ n.source }}</span>
                    <span class="dot">·</span>
                    <span class="chip ghost">{{ prettyTime(n.datetime) }}</span>
                  </div>
                </li>
              </ul>
              <div v-else-if="newsHintLink" class="news-hint" data-testid="news-hint">
                No direct headlines. Try <a :href="newsHintLink" target="_blank" rel="noopener">Google News</a>.
              </div>
              <div v-else class="news-empty">No news available.</div>

              <div v-if="filteredNewsItems.length > newsLimit" class="news-more">
                <button class="btn tiny" @click="newsLimit += 10">Show more</button>
              </div>
            </template>
          </div>

          <div v-if="newsAttribution" class="news-attr">{{ newsAttribution }}</div>
        </template>

        <!-- Tab: Global (by asset class) -->
        <template v-else>
          <div v-if="globalNewsError" class="news-error" role="alert" aria-live="assertive" data-testid="global-news-error">{{ globalNewsError }}</div>

          <!-- Skeleton while loading -->
          <ul v-if="globalNewsLoading" class="news-list rail">
            <li class="news-item rail" v-for="i in 3" :key="'gnsk'+i">
              <div class="skeleton-row"></div>
              <div class="meta-row">
                <span class="chip ghost skeleton-chip"></span>
                <span class="chip ghost skeleton-chip"></span>
              </div>
            </li>
          </ul>

          <div v-else class="news-live" aria-live="polite">
            <template v-if="!globalNewsError">
              <ul v-if="visibleGlobalNews.length" class="news-list rail" data-testid="global-news-list">
                <li v-for="n in visibleGlobalNews" :key="n.id" class="news-item rail">
                  <a class="news-link clamp-2" :href="n.url" target="_blank" rel="noopener">{{ n.headline }}</a>
                  <div class="meta-row">
                    <span class="chip">{{ n.source }}</span>
                    <span class="dot">·</span>
                    <span class="chip ghost">{{ prettyTime(n.datetime) }}</span>
                  </div>
                </li>
              </ul>
              <div v-else class="news-empty">No global headlines available.</div>

              <div v-if="globalNewsItems.length > globalNewsLimit" class="news-more">
                <button class="btn tiny" @click="globalNewsLimit += 10">Show more</button>
              </div>
            </template>
          </div>

          <div v-if="globalNewsAttribution" class="news-attr">{{ globalNewsAttribution }}</div>
        </template>
      </div>

      <!-- Right Overlay (PAIRS only) -->
      <transition name="drop">
        <div
          v-if="rightOverlay.open"
          class="right-overlay"
          :class="{ floating: isCompact }"
          role="dialog"
          aria-modal="true"
          :aria-labelledby="rightOverlay.kind ? (rightOverlay.kind + '-title') : undefined"
          @keydown.esc="closeOverlay"
          tabindex="0"
          ref="overlayEl"
          data-testid="right-overlay"
        >
          <div class="ro-head">
            <div class="ro-title" :id="rightOverlay.kind ? (rightOverlay.kind + '-title') : null">
              <template v-if="rightOverlay.kind==='pairs'">Pair &amp; Selectors</template>
            </div>
            <div class="ro-actions">
              <button class="btn tiny" @click="closeOverlay">Close</button>
            </div>
          </div>

          <div v-if="rightOverlay.kind==='pairs'" class="ro-body">
            <div class="tabs small" style="margin-bottom:8px;">
              <button :class="{ active: activeTab === 'forex' }" @click="setTab('forex')" data-testid="tab-forex">Forex</button>
              <button :class="{ active: activeTab === 'crypto' }" @click="setTab('crypto')" data-testid="tab-crypto">Crypto</button>
            </div>

            <div class="list">
              <template v-if="activeTab === 'forex'">
                <div
                  v-for="p in fxPairs"
                  :key="p.symbol"
                  class="row"
                  @click="selectPair(p, 'fx')"
                  @dblclick="openFullChartNewTab({ key: p.symbol, label: p.symbol, class: 'fx' })"
                >
                  <img class="row-ic" :src="p.icon" :alt="p.symbol" @error="onIconError($event, 'fx')" />
                  <span class="row-name">{{ p.symbol }}</span>
                  <button
                    class="icon-btn pin"
                    :class="{ active: inWatchlist(p, 'fx') }"
                    @click.stop="toggleWatchlist(p, 'fx')"
                    :title="inWatchlist(p, 'fx') ? 'Unpin' : 'Pin'"
                    aria-label="Pin"
                    :aria-pressed="inWatchlist(p, 'fx') ? 'true' : 'false'"
                  >
                    <svg viewBox="0 0 24 24" class="ic"><path d="M14 2l1 5 3 3-5 1-6 8-1-1 8-6 1-5z"/></svg>
                  </button>
                </div>
              </template>

              <template v-else>
                <div
                  v-for="c in cryptoCoins"
                  :key="c.id"
                  class="row"
                  @click="selectPair({ id: c.id, symbol: c.symbol, name: c.name }, 'crypto')"
                  @dblclick="openFullChartNewTab({ key: c.id, label: c.symbol, class: 'crypto' })"
                >
                  <img class="row-ic" :src="c.icon" :alt="c.symbol" @error="onIconError($event, 'crypto')" />
                  <span class="row-name">{{ c.symbol }}</span>
                  <button
                    class="icon-btn pin"
                    :class="{ active: inWatchlist(c, 'crypto') }"
                    @click.stop="toggleWatchlist(c, 'crypto')"
                    :title="inWatchlist(c, 'crypto') ? 'Unpin' : 'Pin'"
                    aria-label="Pin"
                    :aria-pressed="inWatchlist(c, 'crypto') ? 'true' : 'false'"
                  >
                    <svg viewBox="0 0 24 24" class="ic"><path d="M14 2l1 5 3 3-5 1-6 8-1-1 8-6 1-5z"/></svg>
                  </button>
                </div>
              </template>
            </div>
          </div>
        </div>
      </transition>
    </aside>

    <!-- Mount Zeeno once; it manages its own floating window -->
    <KeepAlive>
      <Zeeno v-bind="zeenoProps" />
    </KeepAlive>
  </div>
</template>
<script>
/* eslint-disable no-console */
import { ref, reactive, computed, onMounted, onBeforeUnmount, watch, nextTick, defineAsyncComponent, getCurrentInstance } from 'vue'
import { useRouter } from 'vue-router'
import { signOut } from 'firebase/auth'
import { auth, authReady } from '@/firebase'
import { completeLogout } from '@/router'
import WidgetShell from '@/components/WidgetShell/WidgetShell.vue'
import ChartCard from '@/components/ChartCard/ChartCard.vue' // ⬅️ use ChartCard (emitter)
const Settings = defineAsyncComponent(() => import('@/views/Settings.vue'))
import Zeeno from '@/components/Zeeno/Zeeno.vue'

const tvEnabled = String(import.meta.env.VITE_TV_WIDGETS || '0') === '1'
const probeProvidersEnabled = String(import.meta.env.VITE_PROBE_PROVIDERS || '0') === '1'

import wickwiseEmblem from '@/assets/wickwise-emblem.svg'
import { fetchWithFallback } from '@/services/api.js'

const MAX_CANDLES_POP = Number(import.meta.env.VITE_ZEENO_POP_MAX_CANDLES || 400)

const hscroll = {
  mounted(el) {
    const onWheel = (e) => {
      if (Math.abs(e.deltaY) > Math.abs(e.deltaX)) {
        el.scrollLeft += e.deltaY
        e.preventDefault()
      }
    }
    el.addEventListener('wheel', onWheel, { passive: false })
    el._hscroll = onWheel
  },
  unmounted(el) {
    el.removeEventListener('wheel', el._hscroll)
  }
}

export default {
  name: 'Dashboard',
  components: { WidgetShell, ChartCard, Settings, Zeeno },
  directives: { hscroll },

  errorCaptured(err) {
    try { console.error('[Dashboard errorCaptured]', err) } catch {}
    const fn = (this && (this._setFatal || (this.$data && this.$data._setFatal)))
    if (typeof fn === 'function') {
      fn(err?.message || String(err || 'Unknown error'))
    }
    return false
  },

  setup() {
    // Test-safe router & navigation helpers
    const IS_TEST = typeof import.meta !== 'undefined' && import.meta.env?.MODE === 'test'
    let router = null
    try { router = useRouter() } catch { router = null }

    function safePush(to) {
      try { if (router && typeof router.push === 'function') return router.push(to) } catch {}
      return Promise.resolve()
    }

    const inst = getCurrentInstance()
    const fatalErrorRef = ref('')
    const _setFatal = (msg) => { fatalErrorRef.value = msg }
    if (inst && inst.proxy) { inst.proxy._setFatal = _setFatal }

    // ENV / API
    const backendKey = import.meta.env.VITE_BACKEND_SECRET_KEY || import.meta.env.VITE_BACKEND_KEY || ''
    const apiBase = (import.meta.env.VITE_API_BASE || '/api').replace(/\/+$/, '')
    const marketApiKey = import.meta.env.VITE_MARKET_API_KEY || ''
    const autoRefreshMs = Math.max(60000, Number(import.meta.env.VITE_AUTO_REFRESH_MS || 300000))

    // Providers / attribution
    const serverLiveData = ref(false)
    const providerStatus = reactive(Object.create(null))
    const PROVIDER_META = {
      polygon: { name: 'Polygon.io', icon: '/providers/polygon.svg' },
      finnhub: { name: 'Finnhub', icon: '/providers/finnhub.svg' },
      twelvedata: { name: 'Twelve Data', icon: '/providers/twelvedata.svg' },
      coingecko: { name: 'CoinGecko', icon: '/providers/coingecko.svg' },
      newsapi: { name: 'NewsAPI', icon: '/providers/newsapi.svg' },
      worldbank: { name: 'World Bank', icon: '/providers/worldbank.svg' },
      oanda: { name: 'OANDA', icon: '/providers/oanda.svg' },
      alphavantage: { name: 'Alpha Vantage', icon: '/providers/alphavantage.svg' },
      yahoo: { name: 'Yahoo Finance', icon: '/providers/yahoo.svg' },
      binance: { name: 'Binance', icon: '/providers/binance.svg' },
      coinapi: { name: 'CoinAPI', icon: '/providers/coinapi.svg' },
      cryptocompare: { name: 'CryptoCompare', icon: '/providers/cryptocompare.svg' },
      'wickwise-mock': { name: 'WickWise Mock', icon: '/providers/mock.svg' },
      default: { name: 'Data Provider', icon: '/providers/default.svg' },
    }

    function addProvider(id, ok = true, title = '') {
      const key = String(id || '').toLowerCase().trim()
      if (!key) return
      const prev = providerStatus[key] || {}
      providerStatus[key] = { ok: !!ok, title: title || prev.title || '', last: Date.now() }
    }

    function addProvidersFromText(text) {
      const s = String(text || '').toLowerCase()
      const tests = [
        ['polygon', /polygon/],
        ['finnhub', /finnhub/],
        ['twelvedata', /(twelve[\s-]?data|12data|twelvedata)/],
        ['coingecko', /coingecko/],
        ['newsapi', /newsapi/],
        ['worldbank', /(world\s*bank|wbapi)/],
        ['oanda', /oanda/],
        ['alphavantage', /(alpha[\s-]?vantage|av api)/],
        ['yahoo', /yahoo/],
        ['binance', /binance/],
        ['coinapi', /coinapi/],
        ['cryptocompare', /cryptocompare/],
      ]
      let matched = false
      for (const [id, re] of tests) {
        if (re.test(s)) { addProvider(id, true, text); matched = true }
      }
      if (!matched && s) addProvider('default', true, text)
    }

    // NEW: ingest providers payloads from ChartCard (@providers)
    function ingestProviders(payload) {
      if (!payload) return
      if (Array.isArray(payload)) {
        payload.forEach((p) => {
          if (typeof p === 'string') {
            addProvider(p, true, p)
          } else if (p && (p.id || p.name)) {
            const id = String(p.id || p.name).toLowerCase()
            addProvider(id, p.ok !== false, p.title || p.name || id)
          }
        })
        return
      }
      if (typeof payload === 'string') {
        addProvidersFromText(payload)
        return
      }
      if (payload && typeof payload === 'object') {
        if (Array.isArray(payload.providers)) ingestProviders(payload.providers)
        if (Array.isArray(payload.sources)) ingestProviders(payload.sources)
        const txt = payload.attribution || payload.provider || payload.source || ''
        if (txt) addProvidersFromText(txt)
      }
    }

    async function probeProviders() {
      if (!probeProvidersEnabled) return
      const hasReal = Object.keys(providerStatus).some(k => k !== 'default' && k !== 'wickwise-mock')
      if (hasReal) return

      const candidates = [ '/providers', '/market/providers', '/status/providers', '/about/providers', '/devkey/whoami' ]
      const normId = (s='') => String(s).toLowerCase().trim()

      function ingest(obj) {
        if (!obj) return false
        let any = false
        const addList = (arr) => {
          (arr || []).forEach(x => {
            if (typeof x === 'string') {
              addProvider(normId(x), true, x); any = true
            } else if (x && (x.id || x.name)) {
              const id = normId(x.id || x.name)
              addProvider(id, x.ok !== false, x.title || x.name || id); any = true
            }
          })
        }
        if (Array.isArray(obj)) { addList(obj); return any }
        if (obj.providers) { addList(obj.providers); any = true }
        if (obj.sources)   { addList(obj.sources);   any = true }
        if (obj.keys || obj.config) {
          const bag = obj.keys || obj.config
          Object.keys(bag).forEach(k => {
            const ok = !!bag[k]
            const id = normId(k)
            if (ok) { addProvider(id, true, id); any = true }
          })
        }
        const txt = obj.attribution || obj.provider || obj.source || obj.sourcesText || ''
        if (txt) { addProvidersFromText(txt); any = true }

        if (any && providerStatus['default']) delete providerStatus['default']
        return any
      }

      for (const path of candidates) {
        try {
          const j = await authed(path, { params: { brief: '1' } })
          if (ingest(j)) break
        } catch {}
      }
    }

    const providerChips = computed(() => {
      const ids = Object.keys(providerStatus)
      const chips = ids.map(id => {
        const meta = PROVIDER_META[id] || PROVIDER_META.default
        return {
          id,
          name: meta.name,
          icon: meta.icon,
          ok: !!providerStatus[id]?.ok,
          title: providerStatus[id]?.title || meta.name
        }
      })
      return chips.sort((a,b) => (a.ok === b.ok ? a.name.localeCompare(b.name) : a.ok ? -1 : 1))
    })

    function onProviderIconError(e, p) {
      try {
        if (e?.target?.getAttribute('data-fb')) { e.target.style.display = 'none'; return }
        e.target.src = PROVIDER_META.default.icon
        e.target.setAttribute('data-fb', '1')
      } catch {}
    }

    const serverAuthError = ref('')
    function flagAuthIssue(payload){
      const s = String(payload?.error || payload?.detail || payload?.message || payload || '')
      if (s.includes('api_key') || s.includes('401') || s.includes('403')) {
        serverAuthError.value = 'Client not authorised. Ensure VITE_BACKEND_SECRET_KEY (client) matches BACKEND_SECRET_KEY (server).'
      } else if (s.includes('400')) {
        serverAuthError.value = 'Request rejected by server (400). Check query params (symbol, asset, email) and keys.'
      }
    }

    const authed = (path, options = {}) => {
      const headers = { ...(options.headers || {}) }
      if (backendKey) {
        headers['x-backend-key'] = backendKey
        headers['authorization'] = Bearer ${backendKey}
      }
      const opts = { ...options, headers }
      return fetchWithFallback(path, opts)
        .then((j)=>{ if (j && (j.error||j.detail)) flagAuthIssue(j); return j })
        .catch((e)=>{ flagAuthIssue(e); throw e })
    }

    // ===== THEME =====
    const theme = ref('light'); try { theme.value = localStorage.getItem('theme') || 'light' } catch {}
    const setTheme = (v) => { theme.value = v; try { localStorage.setItem('theme', v) } catch {}; try { document.documentElement.setAttribute('data-theme', v) } catch {}; try { window.dispatchEvent(new Event('themechange')) } catch {} }
    const toggleTheme = () => { setTheme(theme.value === 'dark' ? 'light' : 'dark') }
    const instantToggleTheme = () => { toggleTheme() }
    let disconnectThemeObserver = null
    function initThemeSync(){
      if (typeof window === 'undefined' || typeof document === 'undefined') return
      const apply = () => { try { const t = document.documentElement.getAttribute('data-theme'); if (t === 'light' || t === 'dark') theme.value = t } catch {} }
      apply()
      try {
        const mo = new MutationObserver(apply)
        mo.observe(document.documentElement, { attributes: true, attributeFilter: ['data-theme'] })
        disconnectThemeObserver = () => mo.disconnect()
      } catch { disconnectThemeObserver = null }
    }

    // ===== DEFENSIVE AUTH GUARD =====
    const isAuthenticated = ref(true)
    async function ensureAuth(){
      let ok = true
      try { await authReady; ok = !!(auth && auth.currentUser) } catch { ok = true }
      if (!ok) {
        try { const tok = localStorage.getItem('authToken'); ok = !!tok } catch {}
      }
      isAuthenticated.value = ok
      if (!ok) {
        const redirect = router?.currentRoute?.value?.fullPath || '/dashboard'
        await safePush({ name: 'Login', query: { redirect } }).catch(()=>{})
        await safePush('/login').catch(()=>{})
      }
    }

    // ===== Subscription tier =====
    const devPlan = ref('free')
    const devLevel = ref(0)
    function labelForPlan(p){
      const canon = String(p || '').toLowerCase()
      if (canon === 'owner') return 'Owner'
      if (canon === 'zeeno_plus') return 'Zeeno Plus'
      if (canon === 'zeeno') return 'Zeeno'
      if (canon === 'gold') return 'Gold'
      return 'Free'
    }
    const devPlanLabel = computed(() => labelForPlan(devPlan.value))

    function currentUserEmail(){
      try { const e = auth?.currentUser?.email; if (e) return String(e).trim() } catch {}
      try { const e = localStorage.getItem('userEmail') || localStorage.getItem('email'); if (e) return String(e).trim() } catch {}
      return ''
    }

    async function fetchDevTier(){
      try {
        const email = currentUserEmail()
        if (!email) return
        const j = await authed('/devkey/whoami', { params: { email } })
        if (j && j.ok) {
          devPlan.value = String(j.plan || 'free').toLowerCase()
          const lvl = Number(j.level); devLevel.value = Number.isFinite(lvl) ? lvl : 0
          if (j.keys) {
            Object.keys(j.keys).forEach(k => { if (j.keys[k]) addProvider(k, true, 'Configured') })
          }
        }
      } catch { devPlan.value = 'free'; devLevel.value = 0 }
    }

    // ===== Sidebar/Overlays/Settings =====
    const sidebarCollapsed = ref(false); try { sidebarCollapsed.value = JSON.parse(localStorage.getItem('sidebarCollapsed') || 'false') } catch {}
    const persistSidebar = () => { try { localStorage.setItem('sidebarCollapsed', JSON.stringify(sidebarCollapsed.value)) } catch {} }

    const leftTab = ref('markets')
    const rightOverlay = reactive({ open:false, kind:null })
    const overlayEl = ref(null)

    const marketPanel = reactive({ open:false })
    const marketEl = ref(null)
    const marketsView = ref('list')

    const isCompact = ref(typeof window !== 'undefined' ? window.innerWidth < 1400 : false)
    function onResize(){ isCompact.value = window.innerWidth < 1400 }

    function openOverlay(kind){
      leftTab.value = kind
      rightOverlay.open = true
      rightOverlay.kind = kind
      if (kind === 'pairs') ensurePairsLoaded()
      nextTick(()=> overlayEl.value?.focus())
    }
    function closeOverlay(){ rightOverlay.open = false; rightOverlay.kind = null }

    function openMarketsPanel(){
      leftTab.value = 'markets'
      rightOverlay.open = false
      rightOverlay.kind = null
      marketsView.value = tvEnabled ? 'overview' : 'list'
      marketPanel.open = true
      nextTick(()=> marketEl.value?.focus())
    }
    function closeMarketsPanel(){ marketPanel.open = false }
    function openTop4InPanel(){
      leftTab.value = 'markets'
      marketPanel.open = true
      marketsView.value = 'top4'
      nextTick(loadTop4ForMarket)
    }

    const settingsOpen = ref(false)
    const settingsEl = ref(null)
    function toggleSettings(){
      settingsOpen.value = !settingsOpen.value
      if (settingsOpen.value) {
        leftTab.value = 'settings'
        nextTick(()=> settingsEl.value?.focus())
      } else {
        if (leftTab.value === 'settings') leftTab.value = 'markets'
        try { window.scrollTo({ top: 0, behavior: 'smooth' }) } catch {}
      }
    }
    function onOpenSettingsBridge(){ settingsOpen.value = true; leftTab.value = 'settings'; nextTick(()=> settingsEl.value?.focus()) }

    // Chart type dropdown
    const chartMenuOpen = ref(false)
    function toggleChartTypeMenu(){ chartMenuOpen.value = !chartMenuOpen.value }

    const ui = reactive({
      chartMode: (typeof localStorage !== 'undefined' && localStorage.getItem('ui.chartMode')) || 'candlestick',
      tf: ((typeof localStorage !== 'undefined' && localStorage.getItem('ui.tf')) || '1h').toLowerCase(),
      logScale: JSON.parse((typeof localStorage !== 'undefined' && localStorage.getItem('ui.logScale')) || 'false'),
      show: { volume: JSON.parse((typeof localStorage !== 'undefined' && localStorage.getItem('ui.show.volume')) || 'true') },
      barSpacing: Number((typeof localStorage !== 'undefined' && localStorage.getItem('ui.barSpacing')) || 6)
    })

    function setChartMode(mode){ ui.chartMode = mode; try { localStorage.setItem('ui.chartMode', mode) } catch {}; chartMenuOpen.value = false }
    watch(() => ui.tf, v => { try { localStorage.setItem('ui.tf', v) } catch {} })
    watch(() => ui.logScale, v => { try { localStorage.setItem('ui.logScale', JSON.stringify(v)) } catch {} })
    watch(() => ui.show.volume, v => { try { localStorage.setItem('ui.show.volume', JSON.stringify(v)) } catch {} })
    watch(() => ui.barSpacing, v => { try { localStorage.setItem('ui.barSpacing', String(v)) } catch {} })

    // Named popouts
    let _zeenoPopGate = false
    function openNamedZeenoPopout(url, name = 'WW-Zeeno') {
      if (_zeenoPopGate) return
      _zeenoPopGate = true
      setTimeout(() => { _zeenoPopGate = false }, 700)
      let w = null
      try { w = window.open('', name) } catch {}
      const features = 'popup=1,width=1120,height=760,menubar=no,toolbar=no,location=no,status=no,scrollbars=yes,resizable=yes'
      if (w && !w.closed) { try { w.focus(); w.location.replace(url); return } catch {} }
      window.open(url, name, features)
    }

    let _chartPopGate = false
    function openNamedChartPopout(url, name = 'WW-Chart') {
      if (_chartPopGate) return
      _chartPopGate = true
      setTimeout(() => { _chartPopGate = false }, 700)
      let w = null
      try { w = window.open('', name) } catch {}
      const features = 'popup=1,width=1280,height=820,menubar=no,toolbar=no,location=no,status=no,scrollbars=yes,resizable=yes'
      if (w && !w.closed) { try { w.focus(); w.location.replace(url); return } catch {} }
      window.open(url, name, features + ',noopener')
    }

    function openRouteInNewTabByName(name, query = {}) {
      try {
        const { href } = router.resolve({ name, query })
        if (name === 'FullChart') {
          openNamedChartPopout(href, 'WW-Chart')
        } else if (name === 'ZeenoPopout') {
          openNamedZeenoPopout(href, 'WW-Zeeno')
        } else {
          const w = window.open(href, '_blank', 'noopener,noreferrer')
          if (!w) throw new Error('popup-blocked')
          try { w.opener = null } catch {}
        }
      } catch {
        const qs = new URLSearchParams(query).toString()
        if (name === 'FullChart') {
          openNamedChartPopout('/fullchart' + (qs ? ?${qs} : ''), 'WW-Chart')
        } else {
          const w = window.open((name === 'ZeenoPopout' ? '/zeeno' : '/') + (qs ? ?${qs} : ''), '_blank', 'noopener,noreferrer')
          if (w) try { w.opener = null } catch {}
        }
      }
    }

    function openHomeInNewTab(){
      try {
        const { href } = router.resolve({ path: '/' })
        const w = window.open(href, '_blank', 'noopener,noreferrer')
        if (w) try { w.opener = null } catch {}
      } catch {}
    }

    // === POP-OUT STABILITY PATCH: safe JSON + bounded candles ===
    function safeStringify(obj) { try { return JSON.stringify(obj) } catch { return '' } }
    function byteLen(str='') { try { return new Blob([str]).size } catch { return (str.length || 0) * 2 } }
    function safeStore(key, payload, opts = {}) {
      const maxBytes = Number(opts.maxBytes ?? 1_200_000)
      let p = payload
      const attempts = [
        (x) => x,
        (x) => ({ ...x, candles: Array.isArray(x.candles) ? x.candles.slice(-MAX_CANDLES_POP) : [] }),
        (x) => ({ ...x, candles: Array.isArray(x.candles) ? x.candles.slice(-120) : [] }),
        (x) => ({ ...x, candles: [] }),
      ]
      for (const fn of attempts) {
        const candidate = fn(p)
        const str = safeStringify(candidate)
        if (!str) continue
        if (byteLen(str) <= maxBytes) {
          try { localStorage.setItem(key, str); return true } catch {}
        }
      }
      try { localStorage.setItem(key, safeStringify({ ...payload, candles: [], note: 'trimmed:localStorage-quota' })); return true } catch {}
      return false
    }

    function buildZeenoPopoutPayload(){
      const trimmed = Array.isArray(currentCandles.value) ? currentCandles.value.slice(-MAX_CANDLES_POP) : []
      return {
        selectedPair: { symbol: activeSymbol.value, name: activeName.value || activeSymbol.value, type: activeClass.value },
        timeframe: ui.tf,
        candles: trimmed,
        lastCandle: lastCandle.value,
        subscriptionTier: devPlanLabel.value,
        apiBase,
        userId: userId.value,
        welcomeText: 'Hi! Ask me about this chart or say /full for a full-chart scan.',
        apiKey: marketApiKey,
        serverKey: backendKey || ''
      }
    }
    function persistZeenoPopoutPayload(sid, payload){
      if (!sid) return
      try {
        safeStore(zeeno:popout:${sid}, payload)
        safeStore(wickbot:popout:${sid}, payload)
      } catch {}
    }
    function openZeenoNewTab(){
      const sid = (Math.random().toString(36).slice(2, 8) + Date.now().toString(36)).toUpperCase()
      const payload = buildZeenoPopoutPayload()
      persistZeenoPopoutPayload(sid, payload)
      let href = ''
      try {
        href = router.resolve({
          name: 'ZeenoPopout',
          query: { symbol: activeSymbol.value, type: activeClass.value, timeframe: ui.tf, sid, autoAnalyze: '1', api: apiBase, user: userId.value, theme: theme.value }
        }).href
      } catch {
        const q = new URLSearchParams({
          symbol: activeSymbol.value, type: activeClass.value, timeframe: ui.tf, sid, autoAnalyze: '1', api: apiBase, user: userId.value, theme: theme.value
        }).toString()
        href = '/zeeno?' + q
      }
      openNamedZeenoPopout(href, 'WW-Zeeno')
      closeAllTransientUI()
    }

    // Selection (persist)
    const activeSymbol = ref('EURUSD')
    const activeName = ref('EURUSD')
    const activeClass = ref('fx')
    try {
      const ps = localStorage.getItem('sel.symbol')
      const pc = localStorage.getItem('sel.class')
      const pn = localStorage.getItem('sel.name')
      if (ps && pc) { activeSymbol.value = ps; activeClass.value = pc; activeName.value = pn || ps }
    } catch {}
    watch([activeSymbol, activeClass, activeName], () => {
      try {
        localStorage.setItem('sel.symbol', activeSymbol.value)
        localStorage.setItem('sel.class', activeClass.value)
        localStorage.setItem('sel.name', activeName.value || activeSymbol.value)
      } catch {}
    })
    const selectedLabel = computed(() => activeName.value || activeSymbol.value)

    // Cursor / OHLC
    const ohlc = reactive({ ready:false, open:NaN, high:NaN, low:NaN, close:NaN, volume:NaN, time:null, changePct:0 })
    const lastCandle = ref(null)

    const lastPrevClose = computed(() => {
      const cs = currentCandles.value || []
      const n = cs.length
      const last = cs[n - 1]
      const prev = cs[n - 2]
      return prev ? Number(prev.close) : (last ? Number(last.open) : NaN)
    })

    function updateCursorFromCandle(c){
      if (!c) return
      const prev = Number(lastPrevClose.value)
      const close = Number(c.close)
      const ch = (Number.isFinite(close) && Number.isFinite(prev) && prev !== 0) ? ((close - prev)/prev)*100 : 0
      ohlc.open = c.open; ohlc.high = c.high; ohlc.low = c.low; ohlc.close = c.close; ohlc.volume = c.volume ?? c.v; ohlc.time = c.time
      ohlc.changePct = ch; ohlc.ready = true
    }

    function indexClosestByTime(arr, t){
      if (!arr || !arr.length) return -1
      let lo = 0, hi = arr.length - 1, best = 0
      while (lo <= hi) {
        const mid = (lo + hi) >> 1
        const mt = arr[mid].time
        if (mt === t) { best = mid; break }
        if (mt < t) { best = mid; lo = mid + 1 } else { hi = mid - 1 }
      }
      const a = arr[best]
      const b = arr[Math.min(arr.length - 1, best + 1)]
      return Math.abs((a?.time||0) - t) <= Math.abs((b?.time||0) - t) ? best : Math.min(arr.length - 1, best + 1)
    }

    let raf = 0
    let pendingHover = null
    function applyHoverFromPayload(p){
      if (!p) return
      // Direct OHLC payload
      if (p.open != null && p.high != null && p.low != null && p.close != null) {
        const prev = Number(p.prevClose)
        const c = Number(p.close)
        const ch = (Number.isFinite(c) && Number.isFinite(prev) && prev !== 0) ? ((c - prev)/prev)*100 : 0
        ohlc.open = p.open; ohlc.high = p.high; ohlc.low = p.low; ohlc.close = p.close; ohlc.volume = p.volume; ohlc.time = p.time
        ohlc.changePct = ch; ohlc.ready = true
        return
      }
      // Crosshair time + price
      const t = Number(p.time)
      const cs = currentCandles.value || []
      if (!cs.length || !Number.isFinite(t)) return
      const idx = indexClosestByTime(cs, t)
      if (idx < 0) return
      const bar = cs[idx]
      const prev = cs[idx - 1]
      const close = Number.isFinite(p.priceAtCursor) ? Number(p.priceAtCursor) : Number(bar.close)
      const prevClose = Number(prev?.close)
      const ch = (Number.isFinite(close) && Number.isFinite(prevClose) && prevClose !== 0) ? ((close - prevClose)/prevClose)*100 : 0
      ohlc.open = bar.open; ohlc.high = bar.high; ohlc.low = bar.low; ohlc.close = close; ohlc.volume = bar.volume ?? bar.v; ohlc.time = bar.time
      ohlc.changePct = ch; ohlc.ready = true
    }
    function onCrosshair(payload){
      pendingHover = payload
      if (raf) return
      raf = requestAnimationFrame(() => { raf = 0; const p = pendingHover; pendingHover = null; applyHoverFromPayload(p) })
    }
    function onGlobalCrosshair(e){
      const d = e?.detail || {}
      thead: // (no-op label to keep linters happy)
      {
        const sym = String(d.symbol || '').toUpperCase()
        const cur = String(activeSymbol.value || '').toUpperCase()
        if (!sym || sym !== cur) break thead
        onCrosshair({ time: Number(d.time), priceAtCursor: Number(d.price) })
      }
    }

    watch([() => activeSymbol.value, () => currentCandles.value], () => {
      const lc = currentCandles.value[currentCandles.value.length - 1]
      if (lc) updateCursorFromCandle(lc)
    })

    // Watchlist
    const WATCH_PAGE_SIZE = 14
    const watchlist = ref(JSON.parse(localStorage.getItem('watchlist') || '[]'))
    const syncWatchlist = () => { try { localStorage.setItem('watchlist', JSON.stringify(watchlist.value)) } catch {} }
    const watchlistSorted = computed(() => [...watchlist.value].sort((a,b) => (b.ts || 0) - (a.ts || 0)))
    const watchlistPage = ref(1)
    const watchlistTotalPages = computed(() => Math.max(1, Math.ceil(watchlistSorted.value.length / WATCH_PAGE_SIZE)))
    const pagedWatch = computed(() => {
      const start = (watchlistPage.value - 1) * WATCH_PAGE_SIZE
      const end = start + WATCH_PAGE_SIZE
      return watchlistSorted.value.slice(start, end)
    })
    watch(watchlistSorted, () => { watchlistPage.value = Math.min(watchlistPage.value, watchlistTotalPages.value) })
    const clearWatchlist = () => { watchlist.value = []; syncWatchlist(); watchlistPage.value = 1 }

    function wlKey(type, item) {
      const t = String(type || '').toLowerCase()
      if (t === 'fx') return fx:${String(item.symbol || item.key || '').toUpperCase().replace(/[^A-Z0-9]/g,'')}
      if (t === 'crypto') {
        const base = String(item.id || item.symbol || item.key || '').toLowerCase()
        return crypto:${base}
      }
      return ${t}:${String(item.symbol || item.key || item.label || '').toUpperCase()}
    }
    function wlSymbolForMaps(w) {
      if (w.type === 'fx') return String(w.meta?.symbol || w.label).toUpperCase().replace(/[^A-Z0-9]/g,'')
      if (w.type === 'crypto') return String(w.meta?.id || w.meta?.symbol || w.label).toUpperCase()
      const raw = String(w.meta?.symbol || w.label || '').toUpperCase()
      return raw || String((w.key||'').split(':').at(-1)).toUpperCase()
    }
    const inWatchlist = (item, type) => !!watchlist.value.find(w => w.key === wlKey(type, item))
    const toggleWatchlist = (item, type) => {
      const key = wlKey(type, item)
      const idx = watchlist.value.findIndex((w) => w.key === key)
      if (idx >= 0) { watchlist.value.splice(idx, 1) }
      else { watchlist.value.push({ key, label: item.symbol || item.name || item.label, type, meta: item, ts: Date.now() }) }
      syncWatchlist()
    }
    const addToWatchlist = (w) => { const idx = watchlist.value.findIndex((x) => x.key === w.key); if (idx < 0) { watchlist.value.push({ ...w, ts: Date.now() }); syncWatchlist() } }
    const removeFromWatchlist = (w) => { watchlist.value = watchlist.value.filter((x) => x.key !== w.key); syncWatchlist() }
    const isSelectedWatch = (w) => {
      if (String(w.type).toLowerCase() !== String(activeClass.value).toLowerCase()) return false
      return wlSymbolForMaps(w) === String(activeSymbol.value).toUpperCase()
    }

    const mapGet = (obj, key) => (obj?.[key] ?? obj?.[key?.toUpperCase?.()] ?? obj?.[key?.toLowerCase?.()] ?? null)
    const latestPriceMap = ref({})
    const changeMap = ref({})
    function latestPriceForWatch(w){ return mapGet(latestPriceMap.value, wlSymbolForMaps(w)) }
    function latestChangeForWatch(w){ return mapGet(changeMap.value, wlSymbolForMaps(w)) }

    function quickSelect(w) {
      if (!w) return
      const t = String(w.type || '').toLowerCase()
      if (t === 'fx') { selectPair({ symbol: w.meta?.symbol || w.label }, 'fx'); return }
      if (t === 'crypto') {
        selectPair({ id: w.meta?.id || String(w.meta?.symbol || w.label).toLowerCase(), symbol: w.meta?.symbol || w.label, name: w.meta?.name || w.label }, 'crypto')
        return
      }
      activeSymbol.value = wlSymbolForMaps(w)
      activeName.value = w.label
      activeClass.value = t
      refreshChart()
      refreshNews()
    }

    function openFullChartFromWatchNewTab(w) {
      if (!w) return
      const t = String(w.type || '').toLowerCase()
      const key = (t === 'crypto') ? (w.meta?.id || String(w.meta?.symbol || w.label).toLowerCase()) : String(w.meta?.symbol || w.label).toUpperCase()
      openFullChartNewTab({ key, class: t })
    }

    // ===== Helpers =====
    const sanitizeFx = (s) => String(s || '').toUpperCase().replace(/[^A-Z0-9]/g, '')
    const normIndex = (s) => {
      const u = String(s || '').toUpperCase()
      if (/^(?:\^?GSPC|SPX|SPY)$/.test(u)) return 'SPX'
      if (/^(?:\^?NDX|QQQ)$/.test(u)) return 'NDX'
      if (/^(?:\^?DJI|DIA)$/.test(u)) return 'DJI'
      return u
    }
    const num = (v) => { const n = Number(v); return Number.isFinite(n) ? n : NaN }

    // ===== Dynamic pairs =====
    const fxPairs = ref([])
    const cryptoCoins = ref([])
    const pairsLoadedFx = ref(false)
    const pairsLoadedCrypto = ref(false)

    async function loadFxPairs(){
      try {
        const j = await authed('/market/pairs', { params: { asset: 'fx' } })
        const arr = Array.isArray(j) ? j : (j?.items || j?.pairs || [])
        if (arr.length) {
          fxPairs.value = arr
            .map(x => { const s = sanitizeFx(x.symbol || x.ticker || x.key || x.pair || ''); return { symbol: s, icon: /icons/${s}.png } })
            .filter(p => p.symbol)
          pairsLoadedFx.value = fxPairs.value.length > 0
          return
        }
      } catch {}
      try {
        const t = await authed('/market/top', { params: { asset: 'fx', count: 24 } })
        const items = Array.isArray(t) ? t : (t?.items || [])
        if (items.length) {
          fxPairs.value = items.map(r => {
            const s = sanitizeFx(r.symbol || r.ticker || r.key || r.pair || '')
            return { symbol: s, icon: /icons/${s}.png }
          }).filter(p => p.symbol)
          pairsLoadedFx.value = fxPairs.value.length > 0
          return
        }
      } catch {}
      fxPairs.value = [
        { symbol: 'EURUSD', icon: '/icons/EURUSD.png' },
        { symbol: 'GBPUSD', icon: '/icons/GBPUSD.png' },
        { symbol: 'USDJPY', icon: '/icons/USDJPY.png' },
        { symbol: 'XAUUSD', icon: '/icons/XAUUSD.png' },
      ]
      pairsLoadedFx.value = true
    }

    async function loadCryptoPairs(){
      try {
        const j = await authed('/market/pairs', { params: { asset: 'crypto' } })
        const arr = Array.isArray(j) ? j : (j?.items || j?.pairs || j?.coins || [])
        if (arr.length) {
          cryptoCoins.value = arr.map(x => ({
            id: String(x.id || x.symbol || x.key || '').toLowerCase(),
            symbol: String(x.symbol || x.ticker || x.code || '').toUpperCase(),
            name: x.name || x.symbol || x.id || '',
            icon: /icons/crypto/${String(x.symbol || x.ticker || x.code || '').toUpperCase()}.png
          })).filter(c => c.id && c.symbol)
          pairsLoadedCrypto.value = cryptoCoins.value.length > 0
          return
        }
      } catch {}
      try {
        const t = await authed('/market/top', { params: { asset: 'crypto', count: 24 } })
        const items = Array.isArray(t) ? t : (t?.items || [])
        if (items.length) {
          cryptoCoins.value = items.map(r => ({
            id: String(r.id || r.symbol || r.key || '').toLowerCase(),
            symbol: String(r.symbol || r.ticker || r.key || '').toUpperCase(),
            name: r.name || r.symbol || r.id || '',
            icon: /icons/crypto/${String(r.symbol || r.ticker || r.key || '').toUpperCase()}.png
          })).filter(c => c.id && c.symbol)
          pairsLoadedCrypto.value = cryptoCoins.value.length > 0
          return
        }
      } catch {}
      cryptoCoins.value = [
        { id: 'bitcoin',  symbol: 'BTC', name: 'Bitcoin',  icon: '/icons/crypto/BTC.png' },
        { id: 'ethereum', symbol: 'ETH', name: 'Ethereum', icon: '/icons/crypto/ETH.png' },
        { id: 'solana',   symbol: 'SOL', name: 'Solana',   icon: '/icons/crypto/SOL.png' },
      ]
      pairsLoadedCrypto.value = true
    }

    async function ensurePairsLoaded(){ if (!pairsLoadedFx.value) await loadFxPairs(); if (!pairsLoadedCrypto.value) await loadCryptoPairs() }

    const drawerTabs = ref([
      { key:'indices', label:'Indices' },
      { key:'futures', label:'Futures' },
      { key:'stocks',  label:'Stocks'  },
      { key:'etfs',    label:'ETFs'    },
      { key:'bonds',   label:'Bonds'   },
      { key:'fx',      label:'FX'      },
      { key:'crypto',  label:'Crypto'  }
    ])
    const marketTab = ref('indices')

    const activeTab = ref('forex')
    function setTab(t){ activeTab.value = t }

    function selectPair(p, type) {
      if (type === 'fx') {
        const s = sanitizeFx(p.symbol)
        activeSymbol.value = s; activeName.value = s; activeClass.value = 'fx'
      } else if (type === 'crypto') {
        const id = p.id || (p.symbol ? String(p.symbol).toLowerCase() : '')
        activeSymbol.value = id || p.symbol
        activeName.value = p.symbol || p.name || id
        activeClass.value = 'crypto'
      }
      refreshChart()
      refreshNews()
    }

    const iconFor = (asset, symRaw) => {
      const a = String(asset || '').toLowerCase()
      const s = a === 'fx' ? sanitizeFx(symRaw) : (a === 'indices' ? normIndex(symRaw) : String(symRaw || '').toUpperCase())
      if (a === 'fx') return /icons/${s}.png
      if (a === 'crypto') return /icons/crypto/${s}.png
      if (a === 'indices') return /icons/indices/${s}.png
      if (a === 'stocks') return /icons/stocks/${s}.png
      if (a === 'etfs') return /icons/etfs/${s}.png
      if (a === 'bonds') return /icons/bonds/${s}.png
      if (a === 'futures') return /icons/futures/${s}.png
      return /icons/generic/${a || 'asset'}.png
    }
    const fallbackFor = (asset) => /icons/generic/${String(asset||'asset').toLowerCase()}.png
    const onIconError = (e, asset) => {
      if (!e?.target) return
      const cur = e.target.getAttribute('data-fb')
      if (cur) { e.target.style.display='none'; return }
      e.target.src = fallbackFor(asset)
      e.target.setAttribute('data-fb', '1')
    }

    function normalizeAssetParam(asset) {
      const m = {
        index:'indices', indices:'indices',
        stock:'stocks', stocks:'stocks', equity:'stocks',
        etf:'etfs', etfs:'etfs',
        bond:'bonds', bonds:'bonds', treasury:'bonds',
        future:'futures', futures:'futures',
        forex:'fx', fx:'fx',
        crypto:'crypto'
      }
      const key = String(asset || '').toLowerCase()
      return m[key] || key
    }

    function resolveApiTarget(asset, symbol){
      const fam = normalizeAssetParam(asset)
      if (fam === 'crypto') {
        return { asset: 'crypto', symbol: String(symbol || '').toLowerCase(), id: String(symbol || '').toLowerCase() }
      }
      if (fam === 'fx') {
        return { asset: 'fx', symbol: sanitizeFx(symbol) }
      }
      const symU = String(symbol || '').toUpperCase()
      if (fam === 'indices') {
        if (/^(?:\^?GSPC|SPX|SPY)$/.test(symU)) return { asset: 'etfs', symbol: 'SPY', proxyOf: 'SPX' }
        if (/^(?:\^?NDX|QQQ)$/.test(symU)) return { asset: 'etfs', symbol: 'QQQ', proxyOf: 'NDX' }
        if (/^(?:\^?DJI|DIA)$/.test(symU)) return { asset: 'etfs', symbol: 'DIA', proxyOf: 'DJI' }
      }
      return { asset: fam, symbol: symU }
    }

    const getFetchParams = () => resolveApiTarget(activeClass.value, activeSymbol.value)
    function supportsQuote(asset){ const fam = normalizeAssetParam(asset); return fam === 'stocks' || fam === 'etfs' }

    function fromTopCache(asset, symbol){
      const fam = normalizeAssetParam(asset)
      const symU = String(symbol).toUpperCase()
      const row = (topByClass[fam] || []).find(x => x.symbol === symU || x.key === symU || x.label === symU)
      return row ? { price: row.price, change: row.change } : null
    }

    async function ohlcFallback(symbol, interval, asset, signal){
      try{
        const { asset: fam, symbol: sym } = resolveApiTarget(asset || activeClass.value, symbol)
        const k = await authed('/market/ohlc', { params: { asset: fam, symbol: sym, tf: interval, interval }, signal })
        const arr = Array.isArray(k) ? k : (k?.candles || [])
        const n = arr.length
        const last = arr[n - 1], prev = arr[n - 2]
        if (!last) return { price:null, change:null }
        const price = Number(last.close)
        const ch = (prev && Number(prev.close)) ? ((price - Number(prev.close)) / Number(prev.close)) * 100 : 0
        return { price, change: ch }
      } catch { return { price:null, change:null } }
    }

    // Markets / Top lists
    const topByClass = reactive({ indices:[], stocks:[], etfs:[], bonds:[], futures:[], fx:[], crypto:[] })
    function coerceTopItem(raw, cls) {
      let symbol = raw.symbol ?? raw.ticker ?? raw.pair ?? raw.sym ?? raw.id ?? raw.key ?? raw.name ?? raw.code ?? ''
      let displaySym
      if (cls === 'fx') displaySym = sanitizeFx(symbol)
      else if (cls === 'indices') displaySym = normIndex(symbol)
      else displaySym = String(symbol || '').toUpperCase()

      let price = num(raw.price ?? raw.last ?? raw.close ?? raw.value ?? raw.rate ?? raw.mid ?? raw.mid_price ?? raw.lp ?? raw.c ?? raw.p)
      if (!Number.isFinite(price)) {
        const bid = num(raw.bid), ask = num(raw.ask)
        if (Number.isFinite(bid) && Number.isFinite(ask)) price = (bid + ask) / 2
      }

      let ch = num(raw.change ?? raw.change_pct ?? raw.chg ?? raw.dp ?? raw.percent ?? raw.pct ?? raw.changePercent ?? raw.change_percent)
      if (!Number.isFinite(ch)) {
        const prev = num(raw.prevClose ?? raw.pc ?? raw.previous_close ?? raw.prev ?? raw.prior)
        if (Number.isFinite(prev) && Number.isFinite(price) && prev !== 0) {
          ch = ((price - prev) / prev) * 100
        } else {
          const open = num(raw.open ?? raw.o)
          if (Number.isFinite(open) && Number.isFinite(price) && open !== 0) {
            ch = ((price - open) / open) * 100
          }
        }
      }
      const id = cls === 'crypto' ? (raw.id || String(displaySym || '').toLowerCase()) : undefined
      return { key: displaySym, label: displaySym, price, change: Number.isFinite(ch) ? ch : 0, class: cls, asset: cls, symbol: displaySym, id, priceNum: price, icon: iconFor(cls, displaySym) }
    }

    const DEFAULT_LIST = {
      indices: ['SPX','NDX','DJI'],
      futures: ['ES1!','NQ1!','CL1!'],
      stocks:  ['AAPL','MSFT','NVDA'],
      etfs:    ['SPY','QQQ','IWM'],
      bonds:   ['TLT','IEF','HYG'],
      fx:      ['EURUSD','GBPUSD','USDJPY','XAUUSD'],
      crypto:  ['BTCUSDT','ETHUSDT','SOLUSDT'],
    }
    const HYDRATE_TF = '1h'
    function needsHydrate(it){ return !(Number.isFinite(it?.price) && Number.isFinite(it?.change)) }

    async function fetchOhlcHydrate(cls, sym){
      const target = resolveApiTarget(cls, sym)
      try {
        const d = await authed('/market/ohlc', { params: { asset: target.asset, symbol: target.symbol, tf: HYDRATE_TF, interval: HYDRATE_TF } })
        const arr = Array.isArray(d) ? d : (d?.candles || [])
        const n = arr.length
        const last = arr[n - 1]
        const prev = arr[n - 2]
        if (!last) return null
        const price = Number(last.close)
        const change = (prev && Number(prev.close)) ? ((price - Number(prev.close)) / Number(prev.close)) * 100 : 0
        return coerceTopItem({ symbol: sym, price, change, id: cls==='crypto' ? String(sym).toLowerCase() : undefined }, cls)
      } catch { return null }
    }

    async function hydrateSeedsForClass(cls){
      const seeds = DEFAULT_LIST[cls] || []
      const results = await Promise.allSettled(seeds.map((sym) => fetchOhlcHydrate(cls, sym)))
      return results.map(r => r.status === 'fulfilled' ? r.value : null).filter(Boolean)
    }

    async function fillMissingPricesForClass(cls){
      const list = topByClass[cls] || []
      const jobs = list.map(async (it) => {
        if (!needsHydrate(it)) return it
        const hydrated = await fetchOhlcHydrate(cls, it.id || it.symbol || it.key || it.label)
        if (hydrated) { it.price = hydrated.price; it.change = hydrated.change; it.priceNum = hydrated.price }
        return it
      })
      await Promise.allSettled(jobs)
      topByClass[cls] = (topByClass[cls] || []).filter(row => Number.isFinite(row.price))
    }

    // --- Abort controllers & ids
    let chartAbort = null
    const topAborts = Object.create(null)
    let newsAbort = null
    let tilesAbort = null
    const top4Aborts = Object.create(null)
    let chartReqId = 0

    async function loadTopClass(assetKey) {
      const cls = normalizeAssetParam(assetKey)
      try { topAborts[cls]?.abort?.() } catch {}
      topAborts[cls] = new AbortController()
      try {
        const j = await authed('/market/top', { params: { asset: cls, count: 20 }, signal: topAborts[cls].signal })
        if (j?.attribution || j?.source || j?.provider) addProvidersFromText(j.attribution || j.source || j.provider)
        const items = Array.isArray(j) ? j : (j?.items || j?.top || [])
        topByClass[cls] = (items || []).slice(0, 24).map(r => coerceTopItem(r, cls))
      } catch { topByClass[cls] = [] }

      if (!topByClass[cls]?.length) {
        const hydrated = await hydrateSeedsForClass(cls)
        if (hydrated.length) {
          topByClass[cls] = hydrated
        } else {
          const seeds = DEFAULT_LIST[cls] || []
          topByClass[cls] = seeds.map(sym => {
            const s = cls === 'fx' ? sanitizeFx(sym) : (cls === 'indices' ? normIndex(sym) : String(sym).toUpperCase())
            return { key: s, label: s, price: NaN, change: 0, class: cls, asset: cls, symbol: s, id: cls === 'crypto' ? String(s).toLowerCase() : undefined, icon: iconFor(cls, s) }
          })
        }
      }
      if ((topByClass[cls] || []).some(needsHydrate)) { await fillMissingPricesForClass(cls) }
    }

    async function refreshTop() {
      await Promise.allSettled([
        loadTopClass('indices'),
        loadTopClass('futures'),
        loadTopClass('stocks'),
        loadTopClass('etfs'),
        loadTopClass('bonds'),
        loadTopClass('fx'),
        loadTopClass('crypto'),
      ])

      const seed = (cls) => (topByClass[cls] || []).forEach(it => {
        const keyU = String(it.symbol || it.key || it.label).toUpperCase()
        latestPriceMap.value = { ...latestPriceMap.value, [keyU]: it.price, [keyU.toLowerCase()]: it.price }
        changeMap.value = { ...changeMap.value, [keyU]: it.change, [keyU.toLowerCase()]: it.change }
      })
      ;['indices','futures','stocks','etfs','bonds','fx','crypto'].forEach(seed)

      await refreshTopTiles()
      if (marketsView.value === 'top4') await loadTop4ForMarket()
    }

    const topMovers = computed(() => {
      const all = [
        ...topByClass.indices, ...topByClass.futures, ...topByClass.stocks,
        ...topByClass.etfs,    ...topByClass.bonds,  ...topByClass.fx, ...topByClass.crypto
      ]
      const uniq = new Map()
      for (const x of all) { const k = x.asset+':'+x.symbol; if (!uniq.has(k)) uniq.set(k, x) }
      return [...uniq.values()].sort((a,b)=> Math.abs(b.change)-Math.abs(a.change)).slice(0, 50)
    })

    const headerBestByMarket = computed(() => {
      const order = ['fx','crypto','indices','stocks','etfs','bonds']
      const best = (cls) => {
        const arr = (topByClass[cls] || []).filter(r => Number.isFinite(r.change))
        if (!arr.length) return null
        return arr.reduce((a,b)=> (Number(b.change ?? -Infinity) > Number(a.change ?? -Infinity) ? b : a))
      }
      return order.map(cls => {
        const b = best(cls)
        return b ? { asset: cls, symbol: b.symbol, id: b.id, price: b.price, change: b.change } : null
      }).filter(Boolean)
    })

    const headerMovers = computed(() => {
      const primary = headerBestByMarket.value
      if (primary && primary.length) return primary
      return (topMovers.value || []).slice(0, 6).map(x => ({ asset: x.asset, symbol: x.symbol, id: x.id, price: x.price, change: x.change }))
    })

    const moversBox = computed(() => (topMovers.value || []).slice(0, 8))

    const myTopAssets = reactive([
      { asset: 'indices', symbol: 'SPX' },
      { asset: 'indices', symbol: 'NDX' },
      { asset: 'fx',      symbol: 'EURUSD' },
      { asset: 'crypto',  symbol: 'bitcoin' },
    ])

    const quoteMap = reactive(new Map())
    async function fetchQuote(asset, symbol, signal){
      const fam = normalizeAssetParam(asset)
      if (supportsQuote(fam)) {
        try{
          const j = await authed('/market/quote', { params: { asset: fam, symbol }, signal })
          if (j?.attribution || j?.source || j?.provider) addProvidersFromText(j.attribution || j.source || j.provider)
          const price = Number(j?.price ?? j?.last ?? j?.close ?? NaN)
          const change = Number(j?.change ?? NaN)
          quoteMap.set(asset + ':' + symbol, { price: Number.isFinite(price) ? price : null, change: Number.isFinite(change) ? change : null })
          return
        } catch {}
      }
      const cached = fromTopCache(fam, symbol)
      if (cached){ quoteMap.set(asset + ':' + symbol, cached); return }
      const f = await ohlcFallback(symbol, ui.tf, fam, signal)
      quoteMap.set(asset + ':' + symbol, f)
    }

    async function refreshTopTiles(){
      try { tilesAbort?.abort?.() } catch {}
      tilesAbort = new AbortController()
      await Promise.allSettled(myTopAssets.map(t => fetchQuote(t.asset, t.symbol, tilesAbort.signal)))
    }

    function onTopMoverClick(e, m){
      if (e?.altKey || e?.metaKey || e?.ctrlKey) {
        toggleWatchlist({ symbol: m.symbol, id: m.id || String(m.symbol).toLowerCase(), key: m.symbol, label: m.symbol }, m.asset)
      } else {
        selectFromTop({ asset: m.asset, symbol: m.symbol, id: m.id })
      }
    }
    function selectFromTop(t){
      const cls = t.asset
      if (cls === 'fx')        selectPair({ symbol: t.symbol }, 'fx')
      else if (cls === 'crypto') selectPair({ id: t.id || String(t.symbol).toLowerCase(), symbol: String(t.symbol).toUpperCase(), name: t.symbol }, 'crypto')
      else { activeSymbol.value = t.symbol; activeName.value = t.symbol; activeClass.value = cls; refreshChart(); refreshNews() }
    }

    // Top 4
    const top4Market = ref('fx')
    const marketOptions = [
      { key: 'fx', label: 'FX' },
      { key: 'crypto', label: 'Crypto' },
      { key: 'indices', label: 'Indices' },
      { key: 'stocks', label: 'Stocks' },
      { key: 'etfs', label: 'ETFs' },
      { key: 'bonds', label: 'Bonds' },
      { key: 'futures', label: 'Futures' },
    ]
    const top4DetailMap = reactive(new Map())
    function openTop4FromSidebar(){ openTop4InPanel() }
    const top4Visible = computed(() => {
      const cls = top4Market.value
      const ranked = [...(topByClass[cls] || [])].sort((a,b)=> (b.change??0)-(a.change??0)).slice(0,4)
      return ranked.map(r => {
        const det = top4DetailMap.get(cls+':'+r.symbol) || {}
        return { ...r, high: det.high ?? null, low: det.low ?? null, close: Number.isFinite(det.close) ? det.close : r.price }
      })
    })
    async function loadTop4ForMarket(){
      const cls = top4Market.value
      const ranked = [...(topByClass[cls] || [])].sort((a,b)=> (b.change??0)-(a.change??0)).slice(0,4)
      ranked.forEach(r => { try { top4Aborts[cls+':'+r.symbol]?.abort?.() } catch {} })
      await Promise.allSettled(
        ranked.map(async (r) => {
          try {
            const ab = new AbortController()
            top4Aborts[cls+':'+r.symbol] = ab
            const target = resolveApiTarget(cls, r.id || r.symbol)
            const d = await authed('/market/ohlc', { params: { asset: target.asset, symbol: target.symbol, tf: '1d', interval: '1d' }, signal: ab.signal })
            if (d?.attribution || d?.source || d?.provider) addProvidersFromText(d.attribution || d.source || d.provider)
            const arr = Array.isArray(d) ? d : (d?.candles || [])
            const n = arr.length
            const last = arr[n - 1]
            if (last) { top4DetailMap.set(cls+':'+r.symbol, { high: Number(last.high), low: Number(last.low), close: Number(last.close) }) }
          } catch {}
        })
      )
    }
    function pinTop4(it){ toggleWatchlist({ symbol: it.symbol, id: it.id || String(it.symbol).toLowerCase(), label: it.label }, it.asset) }

    const visibleDrawerItems = computed(() => topByClass[marketTab.value] || [])
    function quickSelectFromChip(item) {
      if (!item) return
      const cls = String(item.class || item.asset || '').toLowerCase()
      if (cls === 'fx') selectPair({ symbol: item.label }, 'fx')
      else if (cls === 'crypto') selectPair({ id: item.id || item.key.toLowerCase(), symbol: item.symbol, name: item.label }, 'crypto')
      else { activeSymbol.value = item.key; activeName.value = item.label; activeClass.value = cls; refreshChart(); refreshNews() }
    }

    // Chart data
    const currentCandles = ref([])
    const reloadChart = ref(0)
    const chartKey = computed(() => ${String(activeClass.value).toLowerCase()}::${String(activeSymbol.value).toUpperCase()}::${ui.tf}::${reloadChart.value})

    const visibleCandles = computed(() => {
      if (!currentCandles.value.length) return []
      const n = zoomWindow.value
      const start = Math.max(0, currentCandles.value.length - n - panOffset.value)
      const end = Math.max(start + 10, currentCandles.value.length - panOffset.value)
      return currentCandles.value.slice(start, end)
    })

    const loading = reactive({ price: false })
    const errors = reactive({ price: '' })
    const marketChartType = computed(() => (ui.chartMode === 'line' ? 'area' : 'candles'))

    function normalizeCandles(raw){
      const out = []
      const seen = new Set()
      for (const k of raw || []) {
        let ts = Number(k.time ?? k.t ?? k.timestamp ?? k.date ?? 0)
        if (Number.isFinite(ts) && ts > 1e12) ts = Math.floor(ts / 1000)
        if (!Number.isFinite(ts)) continue
        if (seen.has(ts)) continue
        seen.add(ts)
        out.push({ time: ts, open: Number(k.open ?? k.o ?? 0), high: Number(k.high ?? k.h ?? 0), low: Number(k.low ?? k.l ?? 0), close: Number(k.close ?? k.c ?? 0), volume:Number(k.volume?? k.v ?? 0) })
      }
      out.sort((a,b)=> a.time - b.time)
      return out
    }

    async function loadChart() {
      const reqId = ++chartReqId
      errors.price = ''
      loading.price = true
      currentCandles.value = []
      lastCandle.value = null
      ohlc.ready = false
      panOffset.value = 0
      try {
        try { chartAbort?.abort?.() } catch {}
        chartAbort = new AbortController()
        const { asset, symbol } = getFetchParams()
        const data = await authed('/market/ohlc', { params: { asset, symbol, tf: ui.tf, interval: ui.tf }, signal: chartAbort.signal })
        if (data?.attribution || data?.source || data?.provider) addProvidersFromText(data.attribution || data.source || data.provider)
        const arrRaw = Array.isArray(data) ? data : (data?.candles || [])
        const arr = normalizeCandles(arrRaw)
        if (reqId !== chartReqId) return
        if (!arr.length) {
          errors.price = 'No candles returned for ' + (symbol || activeSymbol.value) + ' (' + ui.tf + ').'
          addProvider('wickwise-mock', true, 'Candles via fallback/seed')
          return
        }
        currentCandles.value = arr
        const lc = arr[arr.length - 1]
        lastCandle.value = lc || null
        if (lc) updateCursorFromCandle(lc)
        serverLiveData.value = true
      } catch (e) {
        if (reqId === chartReqId) {
          errors.price = e?.message || 'Failed to load chart.'
          currentCandles.value = []
          serverLiveData.value = false
        }
      } finally {
        if (reqId === chartReqId) loading.price = false
      }
    }
    function refreshChart(){ reloadChart.value++; loadChart() }
    watch(() => ui.tf, () => refreshChart())
    watch(() => ui.chartMode, () => { reloadChart.value++ })
    watch(() => [ui.logScale, ui.show.volume, ui.barSpacing], () => { reloadChart.value++ })

    // Zoom & pan
    const ZOOM_WINDOWS = [80, 160, 300, 600, 1000]
    const zoomIndex = ref(2)
    const panOffset = ref(0)
    const zoomWindow = computed(() => ZOOM_WINDOWS[Math.min(ZOOM_WINDOWS.length-1, Math.max(0, zoomIndex.value))])
    function zoomIn(){ zoomIndex.value = Math.max(0, zoomIndex.value - 1); ui.barSpacing = Math.min(14, ui.barSpacing + 2) }
    function zoomOut(){ zoomIndex.value = Math.min(ZOOM_WINDOWS.length - 1, zoomIndex.value + 1); ui.barSpacing = Math.max(3, ui.barSpacing - 2) }
    function panLeft(){ panOffset.value = Math.min((currentCandles.value.length - 10), panOffset.value + Math.round(zoomWindow.value * 0.25)) }
    function panRight(){ panOffset.value = Math.max(0, panOffset.value - Math.round(zoomWindow.value * 0.25)) }
    function resetView(){ zoomIndex.value = 2; panOffset.value = 0; ui.barSpacing = 6 }

    function sma(vals, n){ const out=[]; let s=0; for(let i=0;i<vals.length;i++){ s+=vals[i]; if(i>=n)s-=vals[i-n]; out.push(i>=n-1?s/n:NaN) } return out }

    const aiOverview = computed(() => {
      const cs = currentCandles.value || []
      const closes = cs.map(c=>c.close)
      const ready = closes.length >= 50
      if (!ready) return { ready:false }
      const ma20 = sma(closes,20), ma50 = sma(closes,50)
      const last = closes[closes.length - 1], m20 = ma20[ma20.length - 1], m50 = ma50[ma50.length - 1]
      const slope = (ma20[ma20.length - 1] ?? 0) - (ma20[Math.max(0, ma20.length - 4)] ?? ma20[ma20.length - 1] ?? 0)
      const regimeUp = (m20 || 0) > (m50 || 0)
      const range = cs.slice(-20).map(k => Math.max(k.high-k.low,0))
      const avgRange = range.reduce((a,b)=>a+b,0)/Math.max(range.length,1)
      const meanPx = closes.slice(-20).reduce((a,b)=>a+b,0)/Math.max(20,1)
      const volText = avgRange && meanPx ? (avgRange/meanPx > 0.01 ? 'Volatile' : 'Calm') : '—'
      return {
        ready:true,
        title: regimeUp ? 'Uptrend with mild pullbacks' : 'Down/Sideways regime',
        momentum: slope>0?'Momentum ↑':slope<0?'Momentum ↓':'Momentum →',
        momentumClass: slope>0?'up':slope<0?'down':'flat',
        regime: regimeUp ? 'Trend: Up' : 'Trend: Side/Down',
        regimeClass: regimeUp ? 'up' : 'down',
        volatility: volText,
        summary: Price ${regimeUp ? 'above' : 'near/below'} 50SMA; 20SMA is ${slope>0?'rising':'falling'}.,
        bullets:[
          Last close: ${Number.isFinite(last)?formatPrice(last):'—'},
          MA20: ${Number.isFinite(m20)?formatPrice(m20):'—'},
          MA50: ${Number.isFinite(m50)?formatPrice(m50):'—'}
        ],
        zeenoTake: regimeUp ? 'Dips to MA20 may find buyers.' : 'Rallies into MA20/50 may fade.'
      }
    })

    // ===== News =====
    const newsItems = ref([]), newsError = ref(''), newsHintLink = ref(''), newsAttribution = ref('')
    const newsLoading = ref(false)
    const newsLimit = ref(8)

    // PATCH A — tabs + counters + reload (Relevant/Global)
    const newsTab = ref('relevant')

    // Global news state
    const globalNewsItems = ref([])
    const globalNewsLoading = ref(false)
    const globalNewsError = ref('')
    const globalNewsAttribution = ref('')
    const globalNewsLimit = ref(8)

    function prettyTime(ts){ if(!ts) return ''; const d=new Date(ts*(ts>1e12?1:1000)); return d.toLocaleString() }

    function isRelevantNews(n, symU, nameU){
      const inTickers = Array.isArray(n.tickers) && n.tickers.map(t=>String(t).toUpperCase()).includes(symU)
      if (inTickers) return true
      const h = String(n.headline || '').toUpperCase()
      const sum = String(n.summary || '').toUpperCase()
      if (symU && (h.includes(symU) || sum.includes(symU))) return true
      if (nameU && (h.includes(nameU) || sum.includes(nameU))) return true
      return false
    }

    const filteredNewsItems = computed(() => {
      const symU = String(activeClass.value === 'crypto' ? (activeName.value || activeSymbol.value) : activeSymbol.value).toUpperCase()
      const nameU = String(activeName.value || '').toUpperCase()
      if (!newsItems.value?.length) return []
      const rel = newsItems.value.filter(n => isRelevantNews(n, symU, nameU))
      return rel.length ? rel : newsItems.value
    })
    const visibleNews = computed(() => filteredNewsItems.value.slice(0, newsLimit.value))

    async function refreshNews() {
      try { newsAbort?.abort?.() } catch {}
      newsAbort = new AbortController()
      newsLoading.value = true
      newsError.value = ''
      newsItems.value = []
      newsAttribution.value = ''
      newsHintLink.value = ''
      try {
        const fam = normalizeAssetParam(activeClass.value)
        const sym = resolveApiTarget(activeClass.value, activeSymbol.value).symbol
        let data = null
        if (fam === 'stocks' || fam === 'etfs') {
          data = await authed('/market/news/company', { params: { symbol: sym, limit: 25 }, signal: newsAbort.signal })
        } else {
          data = await authed('/market/news/global', { params: { asset: fam, limit: 25 }, signal: newsAbort.signal })
        }
        if (data?.attribution || data?.source || data?.provider || data?.notes) {
          addProvidersFromText(data.attribution || data.source || data.provider || data.notes)
          newsAttribution.value = [data.attribution, data.notes].filter(Boolean).join(' · ')
        }
        const arr = Array.isArray(data) ? data : (data?.news || data?.items || [])
        newsItems.value = (arr || []).map(n => ({
          id: n.id || n.guid || ${n.source || 'src'}:${n.datetime || n.time || Date.now()},
          url: n.url || n.link || '#',
          headline: n.headline || n.title || '(untitled)',
          source: n.source || 'News',
          datetime: Number(n.datetime || n.time || Date.now()/1000),
          tickers: Array.isArray(n.tickers) ? n.tickers : (Array.isArray(n.symbols) ? n.symbols : []),
          summary: n.summary || n.description || ''
        }))
        if (!newsItems.value.length) {
          const q = (fam === 'fx') ? ${activeSymbol.value} forex : (fam === 'crypto' ? ${activeName.value || activeSymbol.value} crypto : activeSymbol.value)
          newsHintLink.value = https://news.google.com/search?q=${encodeURIComponent(q)}
        } else {
          newsHintLink.value = ''
        }
      } catch (e) {
        newsError.value = e?.message || 'Failed to load news'
      } finally {
        newsLoading.value = false
      }
    }

    // PATCH B — dedicated global news loader (by current asset class)
    async function refreshGlobalNews() {
      globalNewsLoading.value = true
      globalNewsError.value = ''
      globalNewsItems.value = []
      globalNewsAttribution.value = ''
      try {
        const fam = normalizeAssetParam(activeClass.value)
        const data = await authed('/market/news/global', { params: { asset: fam, limit: 25 } })
        if (data?.attribution || data?.source || data?.provider || data?.notes) {
          addProvidersFromText(data.attribution || data.source || data.provider || data.notes)
          globalNewsAttribution.value = [data.attribution, data.notes].filter(Boolean).join(' · ')
        }
        const arr = Array.isArray(data) ? data : (data?.news || data?.items || [])
        globalNewsItems.value = (arr || []).map(n => ({
          id: n.id || n.guid || ${n.source || 'src'}:${n.datetime || n.time || Date.now()},
          url: n.url || n.link || '#',
          headline: n.headline || n.title || '(untitled)',
          source: n.source || 'News',
          datetime: Number(n.datetime || n.time || Date.now()/1000),
          tickers: Array.isArray(n.tickers) ? n.tickers : (Array.isArray(n.symbols) ? n.symbols : []),
          summary: n.summary || n.description || ''
        }))
      } catch (e) {
        globalNewsError.value = e?.message || 'Failed to load global news'
      } finally {
        globalNewsLoading.value = false
      }
    }
    const visibleGlobalNews = computed(() => globalNewsItems.value.slice(0, globalNewsLimit.value))

    // PATCH C — convenience reload for both lists
    async function reloadNewsBoth(){
      await Promise.allSettled([ refreshNews(), refreshGlobalNews() ])
    }

    function openFullChartNewTab(it) {
      const sym = it?.key || it?.id || activeSymbol.value
      const type = (it?.class || it?.asset || activeClass.value) || ''
      try {
        const { href } = router?.resolve?.({ name: 'FullChart', query: { symbol: sym, type, popout: '0' } }) || {}
        if (href) { openNamedChartPopout(href, 'WW-Chart') } else { throw new Error('no-router') }
      } catch {
        const qs = new URLSearchParams({ symbol: sym, type, popout: '0' }).toString()
        openNamedChartPopout('/fullchart?' + qs, 'WW-Chart')
      }
    }
    function openFullChartFromCurrentNewTab() {
      const sym = activeSymbol.value
      const type = activeClass.value
      try {
        const { href } = router?.resolve?.({ name: 'FullChart', query: { symbol: sym, type, popout: '0' } }) || {}
        if (href) { openNamedChartPopout(href, 'WW-Chart') } else { throw new Error('no-router') }
      } catch {
        const qs = new URLSearchParams({ symbol: sym, type, popout: '0' }).toString()
        openNamedChartPopout('/fullchart?' + qs, 'WW-Chart')
      }
    }

    function refreshAll(){ refreshTop(); refreshTopTiles(); refreshChart(); refreshNews() }

    function formatPrice(n){
      if(n==null||!Number.isFinite(n)) return '—'
      if(n>=1000) return n.toFixed(2)
      if(n>=100) return n.toFixed(2)
      if(n>=1) return n.toFixed(3)
      return n.toFixed(5)
    }
    function formatVol(v){ const n=Number(v||0); if(n>=1e9) return (n/1e9).toFixed(2)+'B'; if(n>=1e6) return (n/1e6).toFixed(2)+'M'; if(n>=1e3) return (n/1e3).toFixed(2)+'K'; return n.toFixed(0) }
    function signedPct(n){ const v=Number(n|| 0); return ${v>=0?'+':''}${v.toFixed(2)}% }
    function prettyMarket(k){ const m = { fx:'FX', crypto:'Crypto', indices:'Indices', stocks:'Stocks', etfs:'ETFs', bonds:'Bonds', futures:'Futures' }; return m[k] || k }

    async function logout(){
      try { await signOut(auth) } catch {}
      try { completeLogout() } catch {}
      await safePush({ name: 'Home' }).catch(()=>{})
      await safePush('/').catch(()=>{})
    }

    function closeAllTransientUI(){
      chartMenuOpen.value = false
      if (rightOverlay.open) rightOverlay.open = false
      if (settingsOpen.value) settingsOpen.value = false
      if (marketPanel.open) marketPanel.open = false
    }

    function uuid4() {
      try {
        const g = (typeof globalThis !== 'undefined' && globalThis.crypto) || (typeof window !== 'undefined' && window.crypto)
        if (g && g.getRandomValues) {
          const a = new Uint8Array(16); g.getRandomValues(a)
          a[6] = (a[6] & 0x0f) | 0x40
          a[8] = (a[8] & 0x3f) | 0x80
          const toHex = (n) => n.toString(16).padStart(2, '0')
          const s = Array.from(a, toHex).join('')
          return ${s.slice(0,8)}-${s.slice(8,12)}-${s.slice(12,16)}-${s.slice(16,20)}-${s.slice(20)}
        }
      } catch {}
      return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, c => {
        const r = Math.random() * 16 | 0
        const v = c === 'x' ? r : (r & 0x3) | 0x8
        return v.toString(16)
      })
    }
    const userId = ref(localStorage.getItem('userId') || (localStorage.setItem('userId', uuid4()), localStorage.getItem('userId')))

    function prefetchIcons() {
      const head = document.head || document.getElementsByTagName('head')[0]
      const hrefs = [
        '/icons/EURUSD.png','/icons/GBPUSD.png','/icons/USDJPY.png','/icons/XAUUSD.png',
        '/icons/crypto/BTC.png','/icons/crypto/ETH.png','/icons/crypto/SOL.png'
      ]
      hrefs.forEach(h => { try { const l = document.createElement('link'); l.rel = 'prefetch'; l.as = 'image'; l.href = h; head.appendChild(l) } catch {} })
    }

    function onOpenPricing(e){ const plan = e?.detail?.plan || 'Gold'; console.log('[Pricing] open plan:', plan) }

    const serviceNotice = computed(() => { if (serverAuthError.value) return serverAuthError.value; return '' })

    const zeenoProps = computed(() => ({
      apiKey: marketApiKey,
      selectedPair: { symbol: activeSymbol.value, name: activeName.value || activeSymbol.value },
      timeframe: ui.tf,
      candles: currentCandles.value,
      lastCandle: lastCandle.value,
      subscriptionTier: devPlanLabel.value,
      apiBase,
      userId: userId.value,
      serverKey: backendKey || ''
    }))

    function openZeeno(evt){
      if (evt?.shiftKey || evt?.ctrlKey || evt?.metaKey) { openZeenoNewTab(); return }
      try { localStorage.setItem('zeenoMin','false'); localStorage.setItem('wickbotMin','false') } catch {}
      try { window.dispatchEvent(new Event('zeeno:open')) } catch {}
      closeAllTransientUI()
    }

    const _zeenoEnterEvts = ['zeeno:enlarge','zeeno:expand','zeeno:open-full','zeeno:fullscreen','zeeno:enter-fullscreen','zeeno:open-tab']
    function _zeenoOpenInNewTab (e) { try { e?.preventDefault?.() } catch {}; try { e?.stopPropagation?.() } catch {}; openZeenoNewTab() }

    const rootClasses = computed(() => ({ light: theme.value === 'light', 'sidebar-collapsed': sidebarCollapsed.value }))

    function onZeenoPopinRequest(e){
      try {
        const msg = e?.data
        if (!msg || msg.type !== 'ZEENO_POPIN_REQUEST') return
        if (e.origin !== window.location.origin) return
        const sid = msg.sid || ''
        const payload = buildZeenoPopoutPayload()
        persistZeenoPopoutPayload(sid, payload)
        try { e.source?.postMessage({ type: 'ZEENO_POPIN_PAYLOAD', sid, payload }, e.origin) } catch {}
      } catch {}
    }

    let timer=null
    let unguard = null
    function onKey(e){
      if (e.key === 'Escape') {
        if (rightOverlay.open) closeOverlay()
        if (marketPanel.open) closeMarketsPanel()
        if (chartMenuOpen.value) chartMenuOpen.value = false
        if (settingsOpen.value) settingsOpen.value = false
      }
    }
    function tick(){ try { if (document.hidden) return } catch {}; refreshAll() }

    onMounted(async ()=>{
      initThemeSync()
      try { document.documentElement.setAttribute('data-theme', theme.value) } catch {}
      try { window.dispatchEvent(new Event('themechange')) } catch {}
      if (!IS_TEST) { try { await ensureAuth() } catch {} }
      fetchDevTier()
      probeProviders()
      prefetchIcons()
      refreshAll()
      timer=setInterval(tick, autoRefreshMs)
      window.addEventListener('keydown', onKey)
      window.addEventListener('resize', onResize)
      window.addEventListener('ww:crosshair', onGlobalCrosshair)
      window.addEventListener('open-pricing', onOpenPricing)
      window.addEventListener('dashboard:open-settings', onOpenSettingsBridge)
      _zeenoEnterEvts.forEach(n => window.addEventListener(n, _zeenoOpenInNewTab, { passive: false }))
      try { localStorage.setItem('zeenoMin','false'); localStorage.setItem('wickbotMin','false') } catch {}
      try { window.dispatchEvent(new CustomEvent('zeeno:open')) } catch {}

      // Block in-tab Zeeno/FullChart routes & redirect to popouts
      if (router && typeof router.beforeEach === 'function') {
        unguard = router.beforeEach((to, from, next) => {
          const targetName = String(to?.name || '')
          if (targetName === 'ZeenoPopout') { openZeenoNewTab(); next(false); return }
          if (targetName === 'FullChart' || targetName === 'FullChartDirect') {
            const q = to?.query || {}
            try {
              const { href } = router.resolve({ name: 'FullChart', query: q })
              openNamedChartPopout(href, 'WW-Chart')
            } catch { openFullChartFromCurrentNewTab() }
            next(false); return
          }
          next()
        })
      }
      window.addEventListener('message', onZeenoPopinRequest)
    })

    onBeforeUnmount(()=>{
      if(timer) clearInterval(timer)
      window.removeEventListener('keydown', onKey)
      window.removeEventListener('resize', onResize)
      window.removeEventListener('ww:crosshair', onGlobalCrosshair)
      window.removeEventListener('open-pricing', onOpenPricing)
      window.removeEventListener('dashboard:open-settings', onOpenSettingsBridge)
      _zeenoEnterEvts.forEach(n => window.removeEventListener(n, _zeenoOpenInNewTab))
      try { disconnectThemeObserver?.() } catch {}
      if (raf) cancelAnimationFrame(raf)
      try { chartAbort?.abort?.() } catch {}
      try { newsAbort?.abort?.() } catch {}
      try { tilesAbort?.abort?.() } catch {}
      try { Object.values(topAborts).forEach(a => a?.abort?.()) } catch {}
      try { Object.values(top4Aborts).forEach(a => a?.abort?.()) } catch {}
      try { unguard?.() } catch {}
      window.removeEventListener('message', onZeenoPopinRequest)
    })

    // Keep news in sync with selection
    watch([activeSymbol, activeClass], async () => {
      await refreshNews()
      // Refresh global tab too if user is viewing it, so the counter stays accurate
      if (newsTab.value === 'global') await refreshGlobalNews()
    })

    const changeForSelected = computed(() => {
      const fam = normalizeAssetParam(activeClass.value)
      const key = fam === 'crypto' ? String(activeSymbol.value).toLowerCase() : String(activeSymbol.value).toUpperCase()
      return changeMap.value[key]
    })

    return {
      // state / refs
      fatalError: fatalErrorRef,
      router,
      wickwiseEmblem,
      tvEnabled,
      isAuthenticated,
      theme, toggleTheme, setTheme, instantToggleTheme,
      sidebarCollapsed, persistSidebar,
      leftTab, rightOverlay, openOverlay, closeOverlay, overlayEl,
      settingsOpen, toggleSettings, settingsEl,
      chartMenuOpen, toggleChartTypeMenu,
      ui, setChartMode,
      activeSymbol, activeName, activeClass, selectedLabel,
      ohlc, updateCursorFromCandle,
      watchlist, watchlistSorted, clearWatchlist, inWatchlist, toggleWatchlist, addToWatchlist, removeFromWatchlist, isSelectedWatch,
      quickSelect, openFullChartFromWatchNewTab,
      watchlistPage, watchlistTotalPages, pagedWatch,
      latestPriceForWatch, latestChangeForWatch,
      fxPairs, cryptoCoins, drawerTabs, marketTab, visibleDrawerItems, setTab,
      topMovers, headerBestByMarket, headerMovers, selectFromTop, refreshTop, refreshTopTiles, onTopMoverClick, moversBox,
      currentCandles, visibleCandles, lastCandle, reloadChart, loading, errors, refreshChart, chartKey,
      zoomIn, zoomOut, panLeft, panRight, resetView,
      aiOverview,

      // News (Relevant + Global) — includes A/B/C patches
      newsTab,
      newsItems, newsError, newsHintLink, newsAttribution, newsLoading, prettyTime, refreshNews, filteredNewsItems, visibleNews, newsLimit,
      globalNewsItems, globalNewsLoading, globalNewsError, globalNewsAttribution, refreshGlobalNews, visibleGlobalNews, globalNewsLimit,
      reloadNewsBoth,

      refreshAll, openFullChartNewTab, openFullChartFromCurrentNewTab, openHomeInNewTab, openZeenoNewTab,
      formatPrice, formatVol, signedPct, prettyMarket, iconFor, onIconError,
      top4Market, marketOptions, openTop4FromSidebar, top4Visible, loadTop4ForMarket, pinTop4,
      closeAllTransientUI, logout, quickSelectFromChip,
      marketsView, apiBase, marketApiKey, userId, getFetchParams, resolveApiTarget, normalizeAssetParam,
      isCompact, marketPanel, marketEl, openMarketsPanel, closeMarketsPanel,
      serviceNotice, devPlan, devLevel, devPlanLabel,
      zeenoProps, openZeeno,
      rootClasses, serverLiveData, providerChips, onProviderIconError,
      changeForSelected,
      _setFatal, ensurePairsLoaded,
      marketChartType,

      // handlers for ChartCard events
      onCrosshair,
      ingestProviders,
      addProvidersFromText,
    }
  }
}
</script>

<style scoped>
/* ====== Layout ====== */
.dashboard-root {
  display: grid;
  grid-template-columns: minmax(190px, 220px) minmax(740px, 1fr) minmax(300px, 360px);
  gap: 12px;
  min-height: 100vh;
  background: var(--bg, #0f1419);
  color: #e9eef2;
}
.dashboard-root.light { --bg:#f6f8fb; color:#1b2430; }
.dashboard-root.sidebar-collapsed { grid-template-columns: 64px minmax(720px, 1fr) minmax(300px, 360px); }

@media (max-width: 1340px){
  .dashboard-root { grid-template-columns: minmax(64px, 220px) 1fr; grid-template-areas: "sidebar main" "sidebar right"; }
  .sidebar { grid-area: sidebar; }
  .main { grid-area: main; }
  .right-rail { grid-area: right; }
}
@media (max-width: 1080px){
  .dashboard-root { grid-template-columns: 64px 1fr; }
  .sidebar { width: 64px; }
  .sidebar-scroll { display:none; }
}
@media (max-width: 1180px){ .chart-block { height: 460px; } }

.app-error{
  position: fixed; left: 12px; bottom: 12px; z-index: 1000;
  border:1px solid rgba(255,80,80,.35);
  background: rgba(255,80,80,.12);
  color:#ffd3d3; border-radius:10px; padding:8px 10px; font-size:.9rem; backdrop-filter: blur(6px);
}

.main { display:flex; flex-direction:column; gap:12px; min-height: 0; overflow: auto; min-width: 0; }
.service-notice{ border:1px solid rgba(255,255,255,.18); background:rgba(255,255,255,.06); color:inherit; border-radius:10px; padding:8px 10px; font-weight:700; }

/* ====== Powered by bar ====== */
.powerbar{
  display:flex; align-items:center; gap:10px; flex-wrap:wrap;
  border:1px solid rgba(255,255,255,.12); background:rgba(255,255,255,.04);
  border-radius:12px; padding:8px 10px;
}
.pb-title{ font-weight:900; opacity:.9; font-size:12px; }
.pb-list{ display:flex; gap:6px; align-items:center; list-style:none; margin:0; padding:0; flex-wrap:wrap; }
.pb-chip{
  display:flex; align-items:center; gap:6px;
  border:1px solid rgba(255,255,255,.18);
  background: rgba(255,255,255,.06);
  border-radius:999px; padding:4px 8px; font-size:12px; line-height:1; font-weight:800;
}
.pb-chip.off{ opacity:.7 }
.pb-ic{ width:14px; height:14px; border-radius:4px; object-fit:contain; }
.pb-name{ white-space:nowrap; }
.pb-dot{ display:inline-block; width:6px; height:6px; border-radius:50%; background: rgba(255,255,255,.5); }
.pb-dot.ok{ background:#26a69a } .pb-dot.warn{ background:#efb05e }
.pb-pill{ margin-left:auto; font-size:12px; font-weight:900; border:1px solid rgba(255,255,255,.22); background:linear-gradient(180deg, rgba(255,255,255,.08), rgba(255,255,255,.04)); border-radius:999px; padding:4px 8px; }
.pb-pill.warn{ border-color: rgba(255,120,120,.28); }

/* ====== Settings panel (drop from top) ===== */
.settings-panel{
  border:1px solid rgba(255,255,255,.14); background: rgba(17,24,39,.86);
  backdrop-filter: blur(8px); border-radius: 12px; padding: 10px;
  box-shadow: var(--ww-shadow-2, 0 12px 32px rgba(0,0,0,.45));
}
.dashboard-root.light .settings-panel{ background: rgba(255,255,255,.92); }
.sp-head{ display:flex; align-items:center; justify-content:space-between; margin-bottom:8px; }
.sp-title{ font-weight:900; margin:0; }
.sp-actions{ display:flex; gap:8px; }
.sp-body{ max-height: 70vh; overflow:auto; }
.settings-drop-enter-active, .settings-drop-leave-active { transition: all .18s ease; }
.settings-drop-enter-from, .settings-drop-leave-to { opacity:0; transform: translateY(-10px); }

/* ====== Sidebar ====== */
.sidebar { align-self: start; max-height: 100vh; border:1px solid rgba(255,255,255,0.12); background: rgba(255,255,255,0.04); display: grid; grid-template-rows: auto 1fr; border-radius: 12px; overflow: hidden; }
.sidebar.collapsed { width: 64px; }
.sidebar-top { display:flex; align-items:center; justify-content:space-between; padding:10px; border-bottom:1px solid rgba(255,255,255,.12); }
.brand { display:flex; gap:8px; align-items:center; cursor:pointer; }
.brand-emblem { width:24px; height:24px; }
.brand-text { font-weight: 900; }
.sidebar-scroll { overflow:auto; padding:10px; display:grid; gap:12px; }
.side-tabs { display:flex; flex-direction:column; gap:8px; position:relative; }
.side-tabs button, .side-tab-btn{ display:flex; align-items:center; gap:8px; height:36px; padding:0 10px; border:1px solid rgba(255,255,255,.18); border-radius:999px; background:rgba(255,255,255,.06); color:inherit; cursor:pointer; font-weight:700; font-size:13px; }
.side-tabs button.active, .side-tab-btn.active{ background:rgba(255,255,255,.14) }
.caret{ margin-left:auto; opacity:.7; font-size:11px; }
.tab-with-menu{ position:relative; }
.tab-menu{ position:absolute; top:42px; left:0; z-index:20; background:#111827; border:1px solid rgba(255,255,255,.14); border-radius:10px; padding:6px; display:flex; flex-direction:column; gap:4px; min-width:180px; }
.dashboard-root.light .tab-menu{ background:#ffffff; }
.tab-menu-item{ text-align:left; border:none; background:transparent; color:inherit; padding:6px 8px; border-radius:8px; cursor:pointer; font-weight:700; font-size:12px; }
.tab-menu-item.sel{ background:rgba(255,255,255,.15) }

/* ====== Header ====== */
.topbar{
  position: relative; display:grid; grid-template-columns: 1fr; align-items:center; gap:10px;
  border:1px solid rgba(255,255,255,.12); background:rgba(255,255,255,.04);
  border-radius:12px; padding:6px 8px;
}
.mini-movers{ display:flex; gap:6px; align-items:center; flex-wrap:wrap; }
.mini-title{ opacity:.8; font-weight:800; margin-right:4px; font-size:12px; }
.mini-chip{
  display:flex; gap:4px; align-items:center; border:1px solid rgba(255,255,255,.18); background: transparent;
  border-radius:9999px; padding:2px 6px; cursor:pointer; font-size:11.5px; line-height:1.1;
  transition: background .12s ease, border-color .12s ease, transform .04s ease;
}
.mini-chip:hover{ background:rgba(255,255,255,.07); border-color:rgba(255,255,255,.24) }
.mini-chip:active{ transform: translateY(1px) }
.mini-ic{ width:12px; height:12px; border-radius:4px; opacity:1 }
.mini-market{ font-weight:800; opacity:.8; }
.mini-symbol{ font-weight:900; opacity:.95 }
.mini-chg.up{ color:#26a69a } .mini-chg.down{ color:#ef5350 }
.mini-hint{ opacity:.7; font-size:12px }

/* ====== MARKETS DROP PANEL ====== */
.markets-panel{
  border:1px solid rgba(255,255,255,.14); background: rgba(17,24,39,.86);
  backdrop-filter: blur(8px); border-radius: 12px; padding: 10px;
  box-shadow: var(--ww-shadow-2, 0 12px 32px rgba(0,0,0,.45));
}
.dashboard-root.light .markets-panel{ background: rgba(255,255,255,.96); }
.mp-head{ display:flex; align-items:center; justify-content:space-between; gap:10px; margin-bottom:8px; }
.mp-title{ font-weight:900; margin:0; }
.mp-actions{ display:flex; align-items:center; gap:8px; flex-wrap:wrap; }
.mp-body{ max-height: 64vh; overflow: hidden; }
.mp-overview{ border:1px solid rgba(255,255,255,.14); border-radius:10px; overflow:hidden; background: rgba(0,0,0,.06); }
.dashboard-root.light .mp-overview{ background: rgba(0,0,0,.03); }
.mp-list{ border:1px solid rgba(255,255,255,.14); border-radius:10px; padding:8px; background: rgba(0,0,0,.06); max-height: 64vh; }
.dashboard-root.light .mp-list{ background: rgba(0,0,0,.03); }
.mp-list-inner{ min-width: 760px; }
.markets-drop-enter-active, .markets-drop-leave-active { transition: all .18s ease; }
.markets-drop-enter-from, .markets-drop-leave-to { opacity:0; transform: translateY(-10px); }

/* ====== KPI ====== */
.section{ margin-top:0; }
.kpi-row{ display:grid; grid-template-columns: 1fr 2fr 1fr; gap:8px; }
@media (max-width: 900px){ .kpi-row{ grid-template-columns: 1fr; } }
.kpi-card{ border:1px solid rgba(255,255,255,.12); background:linear-gradient(180deg, rgba(255,255,255,.05), rgba(255,255,255,.03)); border-radius:12px; padding:10px; }
.kpi-card.highlight{ background:linear-gradient(180deg, rgba(255,255,255,.08), rgba(255,255,255,.03)); }
.kpi-title{ font-weight:900; margin-bottom:6px; }
.kpi-selected{ display:flex; align-items:center; justify-content:space-between; }
.kpi-value{ display:flex; align-items:center; gap:8px; font-weight:900; }
.title-ic{ width:14px; height:14px; border-radius:6px; }

/* ====== Chart area ====== */
.chart-section { display:flex; flex-direction:column; }
.chart-controls{
  display:flex; align-items:center; gap:10px; flex-wrap:wrap;
  border:1px solid rgba(255,255,255,.12); background:rgba(255,255,255,.04);
  border-radius:12px; padding:8px 10px; margin:8px 0;
}
.toolbar{ display:flex; gap:6px; }
.cc-select{ padding:6px 8px; }
.cc-check{ display:flex; align-items:center; gap:6px; }
.spacer{ flex:1; }
.chart-block{
  position: relative; border:1px solid rgba(255,255,255,.12);
  background:rgba(255,255,255,.04); border-radius:12px; height: 520px;
  overflow:hidden; min-width: 0;
}
.chart-shell{ position:relative; height:100%; display:flex; }
.chart-shell > * { flex: 1 1 auto; min-width: 0; min-height: 0; }

/* Hide legacy chrome (no-op now) */
.chart-block :deep(.chartcard-head),
.chart-block :deep(.chart-header),
.chart-block :deep(.widget-head),
.chart-block :deep(.chart-settings),
.chart-block :deep(.chart-actions),
.chart-block :deep(.chart-actions .btn),
.chart-block :deep(button[title="Settings"]) {
  display: none !important;
}

/* Overlay buttons on chart */
.chart-overlay{
  position:absolute; top:8px; right:8px; z-index:5; display:flex; gap:6px;
}
.btn.micro{ padding:4px 8px; font-size:.78rem; border-radius:8px; border:1px solid rgba(255,255,255,.18); background:rgba(255,255,255,.06); color:inherit; cursor:pointer; }
.btn.micro.ghost{ background:transparent; border-color:rgba(255,255,255,.22) }
.btn.micro:hover{ background:rgba(255,255,255,.1) }

/* Chart inline error */
.chart-error{
  position:absolute; left:8px; bottom:8px; z-index:6;
  border:1px solid rgba(255,80,80,.35); background: rgba(255,80,80,.12);
  color:#ffd3d3; border-radius:8px; padding:6px 8px; font-size:.85rem;
}

/* OHLC under chart */
.ohlc-card{ border:1px solid rgba(255,255,255,.12); background:rgba(255,255,255,.04); border-radius:12px; padding:8px 10px; margin-top:8px; }
.ohlc-row{ display:grid; grid-template-columns: repeat(6, 1fr); gap:8px; }
@media (max-width: 900px){ .ohlc-row{ grid-template-columns: repeat(3, 1fr); } }
.ohlc-item{ display:flex; flex-direction:column; gap:2px; }
.ohlc-item span{ font-size:12px; opacity:.75; }
.ohlc-item strong{ font-weight:900; }
.ohlc-item strong.up{ color:#26a69a } .ohlc-item strong.down{ color:#ef5350 }

/* ====== Right rail ====== */
.right-rail { display:grid; align-content:start; gap:12px; min-width: 300px; }
.news-card.rail{ border:1px solid rgba(255,255,255,.12); background:linear-gradient(180deg, rgba(255,255,255,.05), rgba(255,255,255,.03)); border-radius:12px; padding:8px; }
.dashboard-root.light .news-card.rail{ background:linear-gradient(180deg, rgba(0,0,0,.03), rgba(0,0,0,.01)); }
.rail-head{ display:flex; align-items:center; justify-content:space-between; margin-bottom:6px; }
.section-title.small{ font-size: 0.95rem; }
.news-list.rail{ list-style:none; padding:0; margin:0; display:grid; gap:6px; }
.news-item.rail{
  display:grid; gap:4px; padding:6px 8px;
  border:1px solid rgba(255,255,255,.06); background: rgba(255,255,255,.02);
  border-radius:10px; transition: background .12s ease, border-color .12s ease, transform .04s ease;
}
.dashboard-root.light .news-item.rail{ background: rgba(0,0,0,.02); border-color: rgba(0,0,0,.06); }
.news-item.rail:hover{ background:rgba(255,255,255,.06); border-color:rgba(255,255,255,.14) }
.dashboard-root.light .news-item.rail:hover{ background:rgba(0,0,0,.06); border-color:rgba(0,0,0,.12) }
.news-link{ color:inherit; text-decoration:none; font-weight:800; line-height:1.25; font-size:.92rem; }
.clamp-2{ display:-webkit-box; -webkit-line-clamp:2; -webkit-box-orient:vertical; overflow:hidden; }
.meta-row{ display:flex; align-items:center; gap:6px; flex-wrap:wrap; font-size:12px; opacity:.9; }
.chip{ display:inline-flex; align-items:center; gap:6px; border:1px solid rgba(255,255,255,.18); background: rgba(255,255,255,.06); padding:2px 6px; border-radius:999px; font-weight:800; line-height:1; }
.chip.ghost{ background: transparent; border-color: rgba(255,255,255,.18); opacity:.9; }
.dashboard-root.light .chip{ background: rgba(0,0,0,.06); border-color: rgba(0,0,0,.14) }
.dashboard-root.light .chip.ghost{ background: transparent; border-color: rgba(0,0,0,.14) }
.news-error{ color:#ef5350; }
.news-attr{ margin-top:6px; opacity:.7; font-size:12px; }
.news-empty{ font-size:.88rem; opacity:.85; padding:6px 2px; }
.news-hint a{ text-decoration:underline; }
.news-more{ margin-top:6px; display:flex; justify-content:flex-end; }

/* Skeleton shimmer */
.skeleton-row{ height: 14px; border-radius:6px; background: linear-gradient(90deg, rgba(255,255,255,.12), rgba(255,255,255,.06), rgba(255,255,255,.12)); background-size:200% 100%; animation: shimmer 1.2s infinite linear; }
.skeleton-chip{ width: 72px; height: 18px; border-radius:999px; display:inline-block; }
@keyframes shimmer{ 0%{ background-position: 200% 0; } 100%{ background-position: -200% 0; } }

/* Watchlist */
.watch-card{ border:1px solid rgba(255,255,255,0.12); background: rgba(255,255,255,0.04); border-radius: 12px; padding: 8px; max-height: 48vh; overflow:auto; }
.empty.small{ font-size: 0.85rem; opacity:.8; }
.watch{ list-style:none; margin:0; padding:0; display:grid; }
.watch.compact{ gap:2px; }
.watch-row{
  display:grid; grid-template-columns: auto 1fr auto auto auto auto; gap:6px; align-items:center;
  padding:6px 8px; border-bottom:1px solid rgba(255,255,255,.08); background:transparent; cursor:pointer;
}
.watch-row:last-child{ border-bottom:none; }

.row-ic{
  width:12px; height:12px; border-radius:9999px; padding:2px 6px;
  background: var(--pill-bg); border:1px solid var(--pill-border); box-shadow: inset 0 0 0 1px var(--pill-glow);
}

.row-name{ font-weight:800; }
.row-price{ opacity:.9; }
.row-chg.up{ color:#26a69a } .row-chg.down{ color:#ef5350 }

.icon-btn.pin{
  width:28px; height:28px; border-radius:999px; border:1px solid rgba(255,255,255,.18);
  background: rgba(255,255,255,.05); display:grid; place-items:center; cursor:pointer;
  transition: filter .15s ease, background .15s ease, border-color .15s ease, transform .15s ease;
}
.icon-btn.pin .ic{ width:14px; height:14px; fill: currentColor; opacity: .9 }
.icon-btn.pin:hover{ background:rgba(255,255,255,.1); border-color:rgba(255,255,255,.28) }
.icon-btn.pin:active{ transform: translateY(1px) }
.icon-btn.pin.active{ background: linear-gradient(180deg, rgba(255,255,255,.18), rgba(255,255,255,.05)); border-color: rgba(255,255,255,.34) }

.wl-x{ border:1px solid rgba(255,255,255,.2); background:rgba(255,255,255,.06); color:inherit; border-radius:8px; padding:4px 6px; cursor:pointer; }

.pager{ display:flex; justify-content:center; align-items:center; gap:8px; }
.pager.compact{ margin-top:6px; }
.pg-btn{ border:1px solid rgba(255,255,255,.2); background:rgba(255,255,255,.06); color:inherit; border-radius:8px; padding:4px 8px; cursor:pointer; }
.pg-ind{ opacity:.85; font-weight:700; font-size:0.9rem; }

/* Right overlay */
.right-overlay{
  position: sticky; top:0; z-index: 10;
  border:1px solid rgba(255,255,255,.14); background: rgba(17,24,39,.72); backdrop-filter: blur(8px);
  border-radius: 12px; padding: 10px; display: grid; grid-template-rows: auto 1fr; outline: none;
  max-height: 80vh; overflow:auto;
}
.right-overlay.floating{
  position: fixed; top: 12px; right: 12px; left: auto; width: min(720px, 96vw);
  max-height: min(90vh, 900px); z-index: 50; box-shadow: 0 20px 60px rgba(0,0,0,.5);
}
.right-overlay .ro-head{ position: sticky; top: 0; background: inherit; z-index: 2; padding-top: 4px; }
.ro-head{ display:flex; align-items:center; justify-content:space-between; margin-bottom:8px; }
.ro-title{ font-weight:900; }
.ro-actions{ display:flex; gap:8px; }
.ro-body{ overflow:auto; }

/* Tabs small */
.tabs.small { display:flex; gap:6px; }
.tabs.small button{
  flex:1; padding:6px 8px; border:1px solid rgba(255,255,255,.18);
  border-radius:8px; background:rgba(255,255,255,.06); color:inherit; cursor:pointer; font-size:12px; font-weight:700;
}
.tabs.small button.active{ background:rgba(255,255,255,.12) }
.tabs.small.scroll{ overflow-x: auto; -webkit-overflow-scrolling: touch; scrollbar-width: thin; padding-bottom: 2px; }
.tabs.small.scroll button{ flex: 0 0 auto; padding: 6px 10px; }
.tabs.small.scroll::-webkit-scrollbar { height: 6px; }
.tabs.small.scroll::-webkit-scrollbar-thumb { background: rgba(255,255,255,.18); border-radius: 4px; }

/* List rows */
.list{ display:grid; gap:2px; }
.row{
  display:grid; grid-template-columns: auto 1fr auto auto auto auto; gap:6px; align-items:center;
  padding:6px 8px; border-bottom:1px solid rgba(255,255,255,.08); background:transparent; cursor:pointer;
}
.row:last-child{ border-bottom:none; }

/* Top 4 overrides */
.top4-row{ display:flex; gap:8px; align-items:center; margin-bottom:8px; }
.top4-label{ font-size:.9rem; opacity:.9; font-weight:800 }
.top4-select{ padding:6px 8px; border-radius:8px; background:#141a22; color:inherit; border:1px solid rgba(255,255,255,.15) }
.dashboard-root.light .top4-select{ background:#fff; }
.top4-grid{ display:grid; grid-template-columns: repeat(2, minmax(0,1fr)); gap:8px; }
.top4-item{ border:1px solid rgba(255,255,255,.14); background:rgba(255,255,255,.06); border-radius:10px; padding:8px; display:grid; gap:6px; cursor:pointer; }
.top4-item .row1{ display:flex; align-items:center; justify-content:space-between; font-weight:900 }
.top4-item .sym-wrap{ display:flex; align-items:center; gap:8px; }
.top4-item .row2{ display:flex; gap:10px; font-size:.9rem; opacity:.95 }
.top4-item .row3{ display:flex; gap:6px; justify-content:flex-end }

/* Overlay transition */
.drop-enter-active, .drop-leave-active { transition: all .18s ease; }
.drop-enter-from, .drop-leave-to { opacity: 0; transform: translateY(-8px); }

/* 2D scroll utility */
.scroll-2d{ overflow: auto; -webkit-overflow-scrolling: touch; }

/* Colors */
.up{ color:#26a69a } .down{ color:#ef5350 }

/* Pill icon aesthetic */
.dashboard-root { --pill-bg: rgba(255,255,255,.06); --pill-border: rgba(255,255,255,.18); --pill-glow: rgba(255,255,255,.06); --pill-pad-x: 6px; --pill-pad-y: 2px; }
.dashboard-root.light { --pill-bg: rgba(0,0,0,.04); --pill-border: rgba(0,0,0,.12); --pill-glow: rgba(0,0,0,.04); }

.mini-ic, .row-ic, .title-ic {
  display: inline-block; width: 12px; height: 12px; object-fit: contain;
  padding: var(--pill-pad-y) var(--pill-pad-x);
  border-radius: 9999px; background: var(--pill-bg);
  border: 1px solid var(--pill-border); box-shadow: inset 0 0 0 1px var(--pill-glow); opacity: 1 !important;
}
.title-ic { width: 14px; height: 14px; }

.mini-chip{ gap: 4px; padding: 2px 6px; font-size: 11.5px; line-height: 1.1; }
.markets-panel .top4-item .row-ic{ padding: var(--pill-pad-y) var(--pill-pad-x); border-radius: 9999px !important; background: var(--pill-bg); border: 1px solid var(--pill-border); box-shadow: inset 0 0 0 1px var(--pill-glow); }

.row, .watch-row { gap: 6px; padding: 6px 8px; }
</style>
