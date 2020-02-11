<script>
  import { afterUpdate, createEventDispatcher } from 'svelte';
  import { _window, matchMq, afterResponsiveUpdate } from '../../stores/responsive.js';
  import ToggleTheme from './ToggleTheme.svelte';
  import { strGeneral, updateGeneral, afterGeneralUpdate } from '../../stores/general.js';
  import { TIMEOUTS } from '../../utils';

  $: currentSection = $strGeneral.pageCurrentSection;
  let pageSections = $strGeneral.pageSections;
  let navPivots = $strGeneral.pageSections.map(section => ({ name: section }));
  let isCalculated = false;
  let wasSelected = null; // when the link is clicked, trigger the fancyBubble
  let isRICScheduled = false;
  let hasReducedMotion = false;

	const dispatch = createEventDispatcher();

  afterGeneralUpdate((prevState, state) => {
    if (!prevState.isReady && state.isReady) {
      setNavigationData();
      window.addEventListener('scroll', handleScroll);
    }

    const prevPageSection = prevState.pageCurrentSection;
    const pageSection = state.pageCurrentSection;

    if (prevPageSection !== pageSection) {
      console.log('pageSection changed:', pageSection);
      if(wasSelected) {
        setTimeout(() => {
          wasSelected = null;
        }, TIMEOUTS.NAV_ANIMATING);
      }
    }

    if (!prevState.isValuesChanging && state.isValuesChanging) {
      console.warn('isValuesChanging - started: scroll to 0');
      /* There are so many animations and layout changes, that the best UX
      is to set the scroll back to the beginning. */
      window.scrollTo(0, 0);
    }

    if (prevState.isValuesChanging && !state.isValuesChanging) {
      console.warn('isValuesChanging - finished: restart nav');
      setNavigationData();
    }
  });
  
  afterResponsiveUpdate((prevState, state) => {
    if(prevState.matchMq.md !== state.matchMq.md) {
    } else {
      setNavigationData();
    }
  })

  function handleScroll() {
    if (isRICScheduled) { return; }

    isRICScheduled = true;

    requestIdleCallback(verifyPageSection, {
      timeout: 150
    });
  }

  function verifyPageSection() {
    isRICScheduled = false;
    const currentY = window.scrollY;

    for (let i = navPivots.length - 1; i >= 0; i--) {
      if (currentY < navPivots[i].y) {
        continue;
      }

      const newCurrentSection = navPivots[i].name;
      if (newCurrentSection !== currentSection) {
        updateGeneral({ pageCurrentSection: newCurrentSection });
      }
      break;
    }
  }

  function getHorizonOffset(scrollY, wHeight) {
    // It means Values is still loading/rendering...
    if (!document.getElementById('nav_valuesEnd')) {
      return 0
    };

    const valuesWidth = (() => {
      // BUG/REVIEW/EDGE_CASE: valuesEnd' parent has a smaller width than its content.
      // Dunno why.... so, instead lets get the position directly from it.
      const valuesEnd = document.getElementById('nav_valuesEnd');
      
      if (!valuesEnd) {
        console.error('Ups! Nav - nav_valuesEnd does not exist! It is a bug!');
        return 0;
      }

      const { left, width } = valuesEnd.getBoundingClientRect();

      // add scrollY in case the resize happens in the "middle" of the page.
      return Math.round(scrollY + left + width);
    })();

    return valuesWidth - wHeight
  }

  function setNavigationData() {
    const isDesktop = $matchMq.md;
    const wWidth = $_window.innerWidth;
    const wHeight = $_window.innerHeight;
    const scrollY = window.scrollY; 
    const horizonOffset = isDesktop ? getHorizonOffset(scrollY, wHeight) : -wHeight*0.5;
    // use this offset to show the section in a better position.
    const dataSection = isDesktop ? 'data-section-offset-h' : 'data-section-offset-v';
    // When recalculating the nav on page resize, we should ignore the horizonSpace
    // so all the math matches correctly. Explaining this is hard.
    const horizonSpaceInPixels = horizonSpace === '100vh' ? wHeight : parseFloat(horizonSpace);
    const anullPrevHorizon = horizonSpaceInPixels - wHeight;

    const newNavPivots = $strGeneral.pageSections.map(section => {
      if (section === 'intro') {
        return { name: 'intro', y: 0 };
      }

      const elSection = document.getElementById(section);
      const sectionTop = scrollY + elSection.getBoundingClientRect().top;

      return {
        name: section,
        y: Math.round(horizonOffset + sectionTop - anullPrevHorizon),
        offset: Math.round((elSection.getAttribute(dataSection) || 0)/100 * wHeight)
      };
    });

    navPivots = newNavPivots;
    console.log('::newNavPivots', newNavPivots)
    isCalculated = true;

    dispatch('calculated', {
      horizonSpace: isDesktop
        // horizonOffset may not exist when isDesktop.
        // it means valuesHorizon isn't loaded yet, so we pass 100vh to avoid
        // showing the other sections while Values is still loading.
        ? horizonOffset ? `${horizonOffset + Math.round(wWidth / 2)}px` : '100vh'
        : 0
		});
  }

  // TODO - handle SR navigation.
  function goToSection(e, pageSection) {
    e.preventDefault();

    if (pageSection === currentSection) {
      return false;
    }

    console.log('pageSection changing to:', pageSection);

    const pivot = navPivots.find(p => p.name === pageSection);
    const to = pageSection !== 'intro' ? pivot.y + pivot.offset : 0;

    wasSelected = pageSection;

    setTimeout(() => {
      console.log('scrolled by click');
      // NOTE: Make sure to call scrollTo before updateGeneral,
      // so all sections read currectly the current scrollY
      window.scrollTo(0, to);

      setTimeout(() => {
        updateGeneral({ pageCurrentSection: pageSection });
      }, TIMEOUTS.NAV_SCROLLED);

    }, TIMEOUTS.NAV_ANIMATING);
  }

  export let horizonSpace;
</script>

<style>
  $itemW: 6rem;

  .nav {
    position: fixed;
    top: $spacer-L;
    left: 50%;
    transform: translateX(-50%);
    z-index: 5; /* above everything */
    display: flex;
    align-items: center;
  }

  :global(.toggleBtn),
  .linksItem {
    /* entry animation */
    opacity: 0;
    $time: 75ms;
  }

  .isReady {
    :global(.toggleBtn),
    .linksItem {
      opacity: 1;
      transition:
        opacity 1000ms var(--delay, $time) cubic-bezier(0.0, 0.0, 0.2, 1),
        transform 1000ms var(--delay, $time) cubic-bezier(0.19, 1, 0.22, 1);
    }

    .linksList::before {
      transform: scale(1);
      transition: transform 700ms cubic-bezier(0.19, 1, 0.22, 1);
    }

    .linksItem {
      &:nth-child(1) { --delay: calc($time*3); }
      &:nth-child(2) { --delay: calc($time*2); }
      &:nth-child(3) { --delay: calc($time*1); }
      &:nth-child(4) { --delay: calc($time*2); }
      &:nth-child(5) { --delay: calc($time*3); }
    }

    :global(.toggleBtn) {
      --delay: calc($time*4);
    }
  }

  .links {
    &List {
      position: relative;
      display: flex;
      margin: 0;
      padding: 0;
      font-size: $font-S;

      &::before {
        content: '';
        position: absolute;
        top: 0;
        left: calc($itemW/2);
        width: calc(100% - $itemW);
        height: 0.1rem;
        background-color: var(--text_invert);
        transform: scale(0, 1);
        pointer-events: none;
      }
    }

    &Item {
      position: relative;
      margin: 0;
      margin-top: calc($spacer-S * -1);
      width: $itemW;
      text-align: center;

      &::before {
        content: '';
        display: block;
        position: absolute;
        top: 0; left: 50%;
        width: $spacer-S;
        height: $spacer-S;
        border-radius: 50%;
        background-color: var(--text_invert);
        transform: translate(-50%, calc($spacer-S/2));
        pointer-events: none;
      }

      &.isCurrent,
      &:hover,
      &:focus-within {
        &::before {
          opacity: 1;
          transition: background-color 400ms;
          background-color: var(--primary_1);
        }
      }
    }

    &Anchor {
      display: block;
      color: var(--text_1);
      padding-top: $spacer-M;
      opacity: 0;
      transition: opacity 400ms;
      text-decoration: none;

      &:hover,
      &:focus {
        outline: none;
        opacity: 1;
        color: var(--primary_1);
      }
    }
  }

  :global(.toggleBtn) {
    margin-top: -3rem;
  }

  .toggleMotion {
    background: transparent;
    border: none;
  }
  .motion {
    width: 1.6rem;
    height: 1.6rem;
    background: var(--text_1);
    border-radius: 50%;
    display: block;
    border: 0.2rem solid var(--bg_0);
    outline: 1px dashed red;
  }

  @media (--md) {
    /* decorative animation */
    .bubble {
      z-index: -1;

      &::before,
      &::after {
        content: '';
        display: block;
        position: absolute;
        top: 0;
        left: 50%;
        width: 200vw;
        height: 200vw;
        border-radius: 50%;
        transform: translate(-50%, -50%) scale(0);
        box-sizing: border-box;
      }

      &::before { background-color: var(--morph_total); }
      &::after { background-color: var(--bg_0); }

      &.wasSelected {
        &::before,
        &::after {
          animation: fancyBubble 1150ms ease-out;
        }

        &::after {
          animation-duration: 1400ms;
        }
      }
    }
  }

  @keyframes fancyBubble {
    0% {
      transform: translate(-50%, -50%) scale(0);
    }
    75% {
      opacity: 1;
    }
    100% {
      transform:translate(-50%, -50%) scale(1);
      opacity: 0;
    }
  }
</style>

<nav class="nav" class:isReady={isCalculated} class:invert={currentSection === 'skills'}>
  <span class="bubble" class:wasSelected={wasSelected}></span>
  <ToggleTheme klass='toggleBtn' />
  <ul class="linksList">
    {#each pageSections as name}
      <li
        class="linksItem"
        class:isCurrent={currentSection === name}>
        <a
          href="#{name}"
          class="linksAnchor"
          aria-current={currentSection === name}
          on:click={e => goToSection(e, name)}>
          {name}
        </a>
      </li>
    {/each}
  </ul>
  <button class="toggleBtn toggleMotion" on:click={() => true} aria-pressed={hasReducedMotion} aria-label="Reduced Motion">
    <span class="motion"></span>
  </button>
</nav>