<script>
  import { Search, GraduationCap, Route, MapPin, X } from 'lucide-svelte';
  import { onMount } from 'svelte';

  let loading = false;
  let loadingCity = false;

  let searchSchoolInput = '';

  let searched = false;
  let searchCityResults = [];
  let searchCityInput = '';
  let searchDistInput = null;

  let parcoursupResults = [];

  let selectedCity = null;

  let tags = [];

  // type 'fili'
  const all_fili = [
    "Autre formation",
    "BTS",
    "BUT",
    "CPGE",
    "EFTS",
    "Ecole d'Ingénieur",
    "Ecole de Commerce",
    "IFSI",
    "Licence",
    "Licence_Las",
    "PASS"
  ];
  
  // type 'acad_mies'
  const academies = [
    "Aix-Marseille",
    "Amiens",
    "Besancon",
    "Bordeaux",
    "Clermont-Ferrand",
    "Corse",
    "Créteil",
    "Dijon",
    "Etranger",
    "Grenoble",
    "Guadeloupe",
    "Guyane",
    "La Réunion",
    "Lille",
    "Limoges",
    "Lyon",
    "Martinique",
    "Mayotte",
    "Montpellier",
    "Nancy-Metz",
    "Nantes",
    "Nice",
    "Normandie",
    "Orléans-Tours",
    "Paris",
    "Poitiers",
    "Polynésie Française",
    "Reims",
    "Rennes",
    "Strasbourg",
    "Toulouse",
    "Versailles"
  ];

  const add_tag = (type, value) => {
    let tag = {
      type: type,
      value: value
    };
    
    // if not already in tags
    if(!tags.find(t => t.type === type && t.value === value)) {
      tags = [...tags, tag];
    }

    searchSchoolInput = '';
  }

  const searchCity = (query) => {
    if (query.length > 2) {
      loadingCity = true;
      fetch(`https://api-adresse.data.gouv.fr/search/?type=municipality&q=${query}&limit=5`)
        .then(response => response.json())
        .then(data => {
          searchCityResults = data.features;
          loadingCity = false;
        })
        .catch(err => {
          console.error(err);
          loadingCity = false;
        });
    }
    else {
      searchCityResults.length = 0;
    }
  }

  const searchCitySelect = (city) => {
    searchCityInput = city.properties.city;
    selectedCity = city;
    if(searchDistInput === null) {
      searchDistInput = 50;
    }
  }

  const removeIfNotSelected = () => {
    if (searchCityInput.trim() == '' || !selectedCity || searchCityInput !== selectedCity.properties.city) {
        searchCityInput = '';
        searchCityResults.length = 0;
      }
  }

  const removeResults = () => {
    // remove markers
    parcoursupResults.forEach(result => {
      map.removeLayer(result.marker);
    });

    // remove results
    parcoursupResults.length = 0;
  }

  const searchParcoursup = (term, city, dist) => {
    loading = true;
    searched = false;
    removeResults();

    if(!city && !term && tags.length === 0) {
      loading = false;
      return;
    }

    let where = `1 = 1`;

    if(term) {
      where += ` AND (form_lib_voe_acc LIKE '%${term}%' OR g_ea_lib_vx LIKE '%${term}%' OR lib_comp_voe_ins LIKE '%${term}%' OR detail_forma LIKE '%${term}%' OR fil_lib_voe_acc LIKE '%${term}%')`;
    }

    if (city && dist) {
      let lat = city.geometry.coordinates[1];
      let lon = city.geometry.coordinates[0];

      where += ` AND within_distance(g_olocalisation_des_formations, geom'POINT(${lon} ${lat})', ${dist}km)`;
    }

    tags.forEach(tag => {
      if(tag.type === 'fili') {
        where += ` AND fili = '${tag.value}'`;
      }
      else if(tag.type === 'academies') {
        where += ` AND academie = '${tag.value}'`;
      }
    });
    

    let url = `https://data.enseignementsup-recherche.gouv.fr/api/explore/v2.1/catalog/datasets/fr-esr-parcoursup/records?where=${where.trim()}&limit=100`;

    fetch(url)
      .then(response => response.json())
      .then(data => {
        // save results
        parcoursupResults = data.results;

        // add markers
        addMarkersOnMap(data.results);

        if(city) {
          // zoom on city
          zoomToCity(city);
        }

        loading = false;
        searched = true;
      })
      .catch(err => {
        console.error(err);
        loading = false;
      });
  }

  const searchInput = () => {
    searchParcoursup(searchSchoolInput, selectedCity, searchDistInput);
  }

  const openFiche = (fiche) => {
    console.log(fiche);
    // open popup
    fiche.marker.openPopup();

    // zoom to marker
    let coords = fiche.marker.getLatLng();
    map.setView([coords.lat, coords.lng], 14);
  }

  let map = null;

  const initMap = () => {
    map = L.map('map').setView([46.3, 1.8], 6)

    L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
      maxZoom: 19,
      attribution: 'Parcoursearch &copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>'
    }).addTo(map);
  }

  const addMarkersOnMap = (results) => {
    results.forEach(result => {
      let lat = result.g_olocalisation_des_formations.lat;
      let lon = result.g_olocalisation_des_formations.lon;

      let marker = L.marker([lat, lon], {
        riseOnHover: true
      }).addTo(map);
      marker.bindPopup(`
        <div class="etab-content">
          <p class="school">${result.g_ea_lib_vx}</p>
          <p class="cursus">${result.fil_lib_voe_acc}</p>
        </div>
      `);
      marker.bindTooltip(result.g_ea_lib_vx);
      marker.setIcon(L.icon({
        iconUrl: '/map-pin.png',
        iconSize: [36, 36],
        iconAnchor: [15, 30],
        popupAnchor: [0, -30]
      }));

      // marker on click, scroll to result
      marker.on('click', () => {
        let resultEl = document.getElementById(`psup-result-${result.cod_aff_form}`);
        resultEl.scrollIntoView({ behavior: 'smooth' });

        setTimeout(() => {
          resultEl.classList.add('highlight');
        }, 500);

        setTimeout(() => {
          resultEl.classList.remove('highlight');
        }, 1000);
      });

      // save marker for later
      result.marker = marker;
    });
  }

  const zoomToCity = (city) => {
    let lat = city.geometry.coordinates[1];
    let lon = city.geometry.coordinates[0];

    map.setView([lat, lon], 12);
  }

  onMount(() => {
    initMap();
  });
</script>

<div class="app">
  <div class="panes-main">
    <div class="pane pane-search">
      <div class="app-title">
        <img class="logo" src="/logo.svg" alt="Parcoursearch" />

        <p class="desc">Recherchez et découvrez les formations Parcoursup de vos rêves avec un moteur de recherche libre.</p>
      </div>
      
      <div class="search-form">
        <div class="search-dropdown-container full">
          <div class="search-input">
            <GraduationCap size="20" />
            <input
              type="text"
              placeholder="Rechercher une formation" 
              autocomplete="off"
              bind:value={searchSchoolInput}
              on:keypress={(e) => e.key === 'Enter' && searchInput()}
            />
          </div>
          <div class="search-dropdown">
            {#if searchSchoolInput.length > 0}
              <div class="search-dropdown-item">
                <Search size="20" />
                <div class="search-dropdown-item-text">
                  <p class="city">{searchSchoolInput}</p>
                  <p class="context">Terme de recherche</p>
                </div>
              </div>
            {:else}
              <div class="search-dropdown-item">
                <p class="tool">Commencez par rechercher une formation</p>
              </div>
            {/if}

            {#if searchSchoolInput.length > 2}
              {#each all_fili.filter(f => f.toLowerCase().includes(searchSchoolInput.toLowerCase())).slice(0, 3) as fil}
                <div
                  class="search-dropdown-item"
                  on:mousedown={() => add_tag('fili', fil) }
                >
                  <GraduationCap size="20" />
                  <div class="search-dropdown-item-text">
                    <p class="city">{fil}</p>
                    <p class="context">Filière</p>
                  </div>
                </div>
              {/each}

              {#each academies.filter(a => a.toLowerCase().includes(searchSchoolInput.toLowerCase())).slice(0, 3) as acad}
                <div class="search-dropdown-item" on:mousedown={() => add_tag('academies', acad)}>
                  <MapPin size="20" />
                  <div class="search-dropdown-item-text">
                    <p class="city">{acad}</p>
                    <p class="context">Académie</p>
                  </div>
                </div>
              {/each}
            {/if}
          </div>
        </div>
        <div class="search-group">

          <div class="search-dropdown-container">
            <div class="search-input">
              <MapPin size="20" />
              <input
                type="text"
                placeholder="Ville" 
                autocomplete="off"
                bind:value={searchCityInput}
                on:input={(e) => searchCity(e.target.value)}
                on:blur={() => removeIfNotSelected()}
              />

              {#if searchCityInput.length > 0}
                <div style="width: 20px; height: 20px;" on:mousedown={() => searchCityInput = ''}>
                  <X style="margin-right: 0px;" size="20" strokeWidth="2.5" />
                </div>
              {/if}
            </div>
            <div class="search-dropdown">
              {#if searchCityResults.length > 0}
                {#each searchCityResults as city}
                  <!-- svelte-ignore a11y-no-static-element-interactions -->
                  <div
                    class="search-dropdown-item"
                    on:mousedown={() => searchCitySelect(city)}
                  >
                    <MapPin size="20" />
                    <div class="search-dropdown-item-text">
                      <p class="city">{city.properties.city}</p>
                      <p class="context">{city.properties.context}</p>
                    </div>
                  </div>
                {/each}
              {:else}
                <div class="search-dropdown-item">
                  {#if loadingCity}
                    <p class="tool">Chargement...</p>
                  {:else if searchCityInput.length > 2}
                    <p class="tool">Aucun résultat pour le terme "{searchCityInput}"</p>
                  {:else}
                    <p class="tool">Commencez par rechercher une ville</p>
                  {/if}

                </div>
              {/if}
            </div>
          </div>

          <div class="search-input input-dist">
            <Route size="20" />
            <input
              type="number"
              placeholder="Dist." 
              autocomplete="off"
                bind:value={searchDistInput}
            />
            <p class="hlpr">km</p>
          </div>
        </div>
        {#if tags.length > 0}
          <div class="tags-container">
            <p class="tags-title">Tags :</p>

            {#each tags as tag}
              <div class="tag">
                <p>{tag.value}</p>
                <div class="tag-close" on:click={() => tags = tags.filter(t => t !== tag)}>
                  <X size="20" />
                </div>
              </div>
            {/each}
          </div>
        {/if}
        <div class="search-button" on:click={searchInput}>
          <Search size="20" />
          <p>Rechercher</p>
        </div>
      </div>

      <p class="cond">
        Ce projet open-source est sous licence libre.<br/>
        2024 Vince Linise - <a href="https://github.com/ecnivtwelve/Parcoursearch">Voir sur GitHub</a>
      </p>

    </div>

    <div class="pane pane-results">
        {#if parcoursupResults.length > 0}
        <div class="parcoursup-results">
          {#each parcoursupResults as result, i}
            <div
              id="psup-result-{result.cod_aff_form}"
              class="parcoursup-result"
              on:click={() => openFiche(result)}
              style="{i < 20 ? `--index: ${i}` : ''}"
            >
              <div class="psup-icon-container">
                <GraduationCap size="20" />
              </div>
              <div class="psup-data">
                <div class="psup-etab-container">
                  <p class="psup-nom-fili">{result.fili}</p>
                  <p class="psup-nom-type">{result.contrat_etab}</p>
                  <p class="psup-nom-etab">{result.g_ea_lib_vx} ({result.dep})</p>
                </div>

              <p class="psup-nom-form">{result.lib_for_voe_ins}</p>
              <p class="psup-nom-ville">{result.dep_lib}, {result.region_etab_aff}</p>
              </div>
            </div>
          {/each}
          </div>
        {:else if loading}
          <div class="loader-container">
            <span class="loader"></span>
          </div>
        {:else if searchSchoolInput.length > 0 && !loading && searched}
          <div class="no-result">
            <Search size="24" />
            <h3>Aucun résultat</h3>
            <p>Essayez de modifier vos critères de recherche.</p>
          </div>
        {:else}
          <div class="no-result">
            <Search size="24" />
            <h3>Commencez par rechercher une formation</h3>
            <p>Utilisez le moteur de recherche de la plate-forme pour trouver des formations Parcoursup.</p>
          </div>
        {/if}

    </div>
  </div>

  <div class="pane pane-map">
    <div id="map"></div>
  </div>
</div>