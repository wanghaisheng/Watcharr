<script lang="ts">
  import { page } from "$app/stores";
  import Activity from "@/lib/Activity.svelte";
  import Error from "@/lib/Error.svelte";
  import HorizontalList from "@/lib/HorizontalList.svelte";
  import Icon from "@/lib/Icon.svelte";
  import PageError from "@/lib/PageError.svelte";
  import PersonPoster from "@/lib/poster/PersonPoster.svelte";
  import Rating from "@/lib/rating/Rating.svelte";
  import SeasonsList from "@/lib/SeasonsList.svelte";
  import Spinner from "@/lib/Spinner.svelte";
  import Status from "@/lib/Status.svelte";
  import ProvidersList from "@/lib/content/ProvidersList.svelte";
  import SimilarContent from "@/lib/content/SimilarContent.svelte";
  import Title from "@/lib/content/Title.svelte";
  import VideoEmbedModal from "@/lib/content/VideoEmbedModal.svelte";
  import { contentExistsOnJellyfin, removeWatched, updateWatched } from "@/lib/util/api";
  import { getTopCrew } from "@/lib/util/helpers.js";
  import { serverFeatures, userSettings, watchedList } from "@/store";
  import type {
    TMDBContentCredits,
    TMDBContentCreditsCrew,
    TMDBShowDetails,
    WatchedStatus
  } from "@/types";
  import axios from "axios";
  import { onMount } from "svelte";
  import RequestShow from "@/lib/request/RequestShow.svelte";
  import FollowedThoughts from "@/lib/content/FollowedThoughts.svelte";
  import ArrRequestButton from "@/lib/request/ArrRequestButton.svelte";
  import tooltip from "@/lib/actions/tooltip.js";
  import MyThoughts from "@/lib/content/MyThoughts.svelte";
  import AddToTagButton from "@/lib/tag/AddToTagButton.svelte";

  $: settings = $userSettings;

  export let data;

  let trailer: string | undefined;
  let trailerShown = false;
  let requestModalShown = false;
  let jellyfinUrl: string | undefined;
  let arrRequestButtonComp: ArrRequestButton;

  $: wListItem = $watchedList.find(
    (w) => w.content?.type === "tv" && w.content?.tmdbId === data.tvId
  );

  let showId: number | undefined;
  let show: TMDBShowDetails | undefined;
  let pageError: Error | undefined;

  onMount(() => {
    const unsubscribe = page.subscribe((value) => {
      console.log(value);
      const params = value.params;
      if (params && params.id) {
        showId = Number(params.id);
      }
    });
    return unsubscribe;
  });

  $: {
    (async () => {
      try {
        show = undefined;
        pageError = undefined;
        if (!showId) {
          return;
        }
        const data = (
          await axios.get(`/content/tv/${showId}`, { params: { region: settings?.country } })
        ).data as TMDBShowDetails;
        if (data.videos?.results?.length > 0) {
          const t = data.videos.results.find((v) => v.type?.toLowerCase() === "trailer");
          if (t?.key) {
            if (t?.site?.toLowerCase() === "youtube") {
              trailer = `https://www.youtube.com/embed/${t?.key}`;
            }
          }
        }
        contentExistsOnJellyfin("tv", data.name, data.id).then((j) => {
          if (j?.hasContent && j?.url !== "") {
            jellyfinUrl = j.url;
          }
        });
        show = data;
      } catch (err: any) {
        show = undefined;
        pageError = err;
      }
    })();
  }

  async function getTvCredits() {
    const credits = (await axios.get(`/content/tv/${data.tvId}/credits`))
      .data as TMDBContentCredits & { topCrew: TMDBContentCreditsCrew[] };
    if (credits.crew?.length > 0) {
      credits.topCrew = getTopCrew(credits.crew);
    }
    return credits;
  }

  async function contentChanged(
    newStatus?: WatchedStatus,
    newRating?: number,
    newThoughts?: string,
    pinned?: boolean
  ): Promise<boolean> {
    if (!data.tvId) {
      console.error("contentChanged: no tvId");
      return false;
    }
    return await updateWatched(data.tvId, "tv", newStatus, newRating, newThoughts, pinned);
  }
</script>

<svelte:head>
  <title>{show?.name ? `${show.name} - ` : ""}Show</title>
</svelte:head>

{#if pageError}
  <PageError pretty="Failed to load tv show!" error={pageError} />
{:else if !show}
  <Spinner />
{:else if Object.keys(show).length > 0}
  <div>
    <div class="content">
      {#if show?.backdrop_path}
        <img
          class="backdrop"
          src={"https://www.themoviedb.org/t/p/w1920_and_h800_multi_faces" + show.backdrop_path}
          alt=""
        />
      {/if}
      <div class="vignette" />

      <div class="details-container">
        <img class="poster" src={"https://image.tmdb.org/t/p/w500" + show.poster_path} alt="" />

        <div class="details">
          <Title
            title={show.name}
            homepage={show.homepage}
            releaseYear={new Date(Date.parse(show.first_air_date)).getFullYear()}
            voteAverage={show.vote_average}
            voteCount={show.vote_count}
          />

          <span class="quick-info">
            {#if show?.episode_run_time?.length > 0}
              <span>{show.episode_run_time.join(",")} min</span>
            {/if}

            <div>
              {#each show.genres as g, i}
                <span>{g.name}{i !== show.genres.length - 1 ? ", " : ""}</span>
              {/each}
            </div>
          </span>

          <span style="font-weight: bold; font-size: 14px;">Overview</span>
          <p>{show.overview}</p>

          <div class="btns">
            {#if trailer}
              <button on:click={() => (trailerShown = !trailerShown)}>View Trailer</button>
              {#if trailerShown}
                <VideoEmbedModal embed={trailer} closed={() => (trailerShown = false)} />
              {/if}
            {/if}
            {#if jellyfinUrl}
              <a class="btn" href={jellyfinUrl} target="_blank">
                {#if localStorage.getItem("useEmby")}
                  <Icon i="emby" wh={14} />Play On Emby
                {:else}
                  <Icon i="jellyfin" wh={14} />Play On Jellyfin
                {/if}
              </a>
            {/if}
            {#if $serverFeatures.sonarr && data.tvId}
              <ArrRequestButton
                type="tv"
                tmdbId={data.tvId}
                openRequestModal={() => (requestModalShown = !requestModalShown)}
                bind:this={arrRequestButtonComp}
              />
            {/if}
            {#if wListItem}
              <div class="other-side">
                <AddToTagButton watchedItem={wListItem} />
                <button
                  on:click={() => {
                    if (wListItem?.pinned) {
                      contentChanged(undefined, undefined, undefined, false);
                    } else {
                      contentChanged(undefined, undefined, undefined, true);
                    }
                  }}
                  use:tooltip={{
                    text: `${wListItem?.pinned ? "Unpin from" : "Pin to"} top of list`,
                    pos: "bot"
                  }}
                >
                  <Icon i={wListItem?.pinned ? "unpin" : "pin"} wh={19} />
                </button>
                <button
                  class="delete-btn"
                  on:click={() =>
                    wListItem
                      ? removeWatched(wListItem.id)
                      : console.error("no wlistItem.. can't delete")}
                  use:tooltip={{ text: "Delete", pos: "bot" }}
                >
                  <Icon i="trash" wh={19} />
                </button>
              </div>
            {/if}
          </div>

          <ProvidersList providers={show["watch/providers"]} />
        </div>
      </div>
    </div>

    {#if requestModalShown}
      <RequestShow
        content={show}
        onClose={(reqResp) => {
          requestModalShown = false;
          if (reqResp) {
            arrRequestButtonComp.setExistingRequest(reqResp);
          }
        }}
      />
    {/if}

    <div class="page">
      <div class="review">
        <!-- <span>What did you think?</span> -->
        <Rating rating={wListItem?.rating} onChange={(n) => contentChanged(undefined, n)} />
        <Status status={wListItem?.status} onChange={(n) => contentChanged(n)} />
        {#if wListItem}
          <MyThoughts
            contentTitle={show.name}
            thoughts={wListItem?.thoughts}
            onChange={(newThoughts) => {
              return contentChanged(undefined, undefined, newThoughts);
            }}
          />
        {/if}
      </div>

      {#if showId}
        <FollowedThoughts mediaType="tv" mediaId={showId} />
      {/if}

      {#await getTvCredits()}
        <Spinner />
      {:then credits}
        {#if credits.topCrew?.length > 0}
          <div class="creators">
            {#each credits.topCrew as crew}
              <div>
                <span>{crew.name}</span>
                <span>{crew.job}</span>
              </div>
            {/each}
          </div>
        {/if}

        {#if credits.cast?.length > 0}
          <HorizontalList title="Cast">
            {#each credits.cast?.slice(0, 50) as cast}
              <PersonPoster
                id={cast.id}
                name={cast.name}
                path={cast.profile_path}
                role={cast.character}
                zoomOnHover={false}
              />
            {/each}
          </HorizontalList>
        {/if}
      {:catch err}
        <Error error={err} pretty="Failed to load cast!" />
      {/await}

      <SimilarContent type="tv" similar={show.similar} />

      {#if wListItem}
        <Activity wListId={wListItem.id} activity={wListItem.activity} />
      {/if}
      {#if data?.tvId}
        <SeasonsList tvId={data.tvId} seasons={show.seasons} watchedItem={wListItem} />
      {/if}
    </div>
  </div>
{:else}
  Show not found
{/if}

<style lang="scss">
  .content {
    position: relative;
    color: white;

    img.provider {
      width: 45px;
      height: 45px;
      box-shadow: 0px 0px 8px -4px #9c8080;
      border-radius: 50px;
    }

    img.backdrop {
      position: absolute;
      left: 0;
      top: 0;
      z-index: -2;
      width: 100%;
      height: 100%;
      object-fit: cover;
      filter: $backdrop-filter;
      mix-blend-mode: $backdrop-mix-blend-mode;
      mask-image: $backdrop-mask-image;
    }

    .vignette {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      background-color: rgba($color: #000000, $alpha: 0.7);
      z-index: -1;
      mask-image: $backdrop-mask-image;
    }

    .details-container {
      display: flex;
      flex-flow: row;
      gap: 35px;
      max-width: 1100px;
      padding: 40px 80px;
      margin-left: auto;
      margin-right: auto;

      img.poster {
        width: 235px;
        height: 100%;
        box-shadow: 0px 0px 14px -4px #9c8080;
        border-radius: 12px;
      }

      .details {
        display: flex;
        flex-flow: column;
        gap: 5px;

        .quick-info {
          display: flex;
          gap: 10px;
          margin-bottom: 8px;
        }

        p {
          font-size: 14px;
          margin-bottom: 18px;
        }

        .btns {
          display: flex;
          flex-flow: row;
          flex-wrap: wrap;
          gap: 8px;
          margin-top: auto;

          a.btn,
          button {
            max-width: fit-content;
            overflow: hidden;
            animation: 50ms cubic-bezier(0.86, 0, 0.07, 1) forwards otherbtn;
            white-space: nowrap;
            gap: 6px;
            justify-content: flex-start;
            font-size: 14px;

            @keyframes otherbtn {
              from {
                width: 0px;
              }
              to {
                width: 100%;
              }
            }
          }

          .other-side {
            display: flex;
            flex-flow: row;
            gap: 8px;

            @media screen and (min-width: 900px) {
              margin-left: auto;
            }
          }

          .delete-btn {
            &:hover {
              color: $error;
            }
          }
        }
      }

      @media screen and (max-width: 700px) {
        padding: 40px;
      }

      @media screen and (max-width: 590px) {
        flex-flow: column;
        align-items: center;
      }
    }
  }

  .page {
    display: flex;
    flex-flow: column;
    align-items: center;
    margin-left: auto;
    margin-right: auto;
    gap: 30px;
    padding: 20px 50px;
    max-width: 1200px;

    @media screen and (max-width: 500px) {
      padding: 20px;
    }
  }

  .review {
    display: flex;
    flex-flow: column;
    gap: 10px;
    width: 100%;
    max-width: 380px;

    @media screen and (max-width: 420px) {
      max-width: 340px;
    }
  }

  .creators {
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    gap: 35px;
    margin: 10px 60px;

    div {
      display: flex;
      flex-flow: column;
      min-width: 150px;

      span:first-child {
        font-weight: bold;
      }
    }
  }
</style>
