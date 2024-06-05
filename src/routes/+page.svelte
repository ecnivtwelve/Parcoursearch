<script>
  import { Search, GraduationCap, Route, MapPin } from 'lucide-svelte';
</script>

<div class="app">
  <div class="pane pane-search">
    
    <div class="search-form">
      <div class="search-input">
        <GraduationCap size="20" />
        <input
          type="text"
          placeholder="Rechercher une formation" 
          autocomplete="off"
        />
      </div>
      <div class="search-group">
        <div class="search-input">
          <MapPin size="20" />
          <input
            type="text"
            placeholder="Ville" 
            autocomplete="off"
          />
        </div>
        <div class="search-input input-dist">
          <Route size="20" />
          <input
            type="number"
            placeholder="Dist." 
            autocomplete="off"
          />
          <p class="hlpr">km</p>
        </div>
      </div>
      <div class="search-button">
        <Search size="20" />
        <p>Rechercher</p>
      </div>
    </div>

  </div>

  <div class="pane pane-results">
  
  </div>

  <div class="pane pane-map">
  
  </div>
</div>