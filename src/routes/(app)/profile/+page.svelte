<script lang="ts">
  import { goto } from "$app/navigation";
  import Checkbox from "@/lib/Checkbox.svelte";
  import Error from "@/lib/Error.svelte";
  import Spinner from "@/lib/Spinner.svelte";
  import Setting from "@/lib/settings/Setting.svelte";
  import Stat from "@/lib/stats/Stat.svelte";
  import Stats from "@/lib/stats/Stats.svelte";
  import { updateUserSetting } from "@/lib/util/api";
  import { getOrdinalSuffix, monthsShort, toggleTheme } from "@/lib/util/helpers";
  import { appTheme, userInfo, userSettings } from "@/store";
  import { UserType, type Image, type Profile } from "@/types";
  import axios from "axios";
  import { notify } from "@/lib/util/notify";
  import UserAvatar from "@/lib/img/UserAvatar.svelte";
  import PwChangeModal from "@/routes/(app)/profile/modals/PwChangeModal.svelte";
  import SyncModal from "./modals/SyncModal.svelte";
  import RegionDropDown from "@/lib/RegionDropDown.svelte";
  import RatingSetting from "@/lib/rating/RatingSetting.svelte";

  $: user = $userInfo;
  $: settings = $userSettings;
  $: selectedTheme = $appTheme;

  let privateDisabled = false;
  let privateThoughtsDisabled = false;
  let exportDisabled = false;
  let hideSpoilersDisabled = false;
  let countryDisabled = false;
  let includePreviouslyWatchedDisabled = false;
  let automateShowStatusesDisabled = false;
  let pwChangeModalOpen = false;
  let getProfilePromise = getProfile();
  let jellyfinSyncModalOpen = false;
  let plexSyncModalOpen = false;

  async function getProfile() {
    return (await axios.get(`/profile`)).data as Profile;
  }

  function formatDate(d: Date) {
    return `${d.getDate()}${getOrdinalSuffix(d.getDate())} ${
      monthsShort[d.getMonth()]
    } ${d.getFullYear()}`;
  }

  function updateBio(ev: FocusEvent & { currentTarget: EventTarget & HTMLTextAreaElement }) {
    const newBio = ev?.currentTarget?.value;
    if (typeof newBio !== "string") {
      console.warn("updateBio called without any value", newBio);
      return;
    }
    const nid = notify({ text: "Updating Bio", type: "loading" });
    axios
      .post("/user/bio", { newBio: newBio })
      .then(() => {
        if (user) {
          user.bio = newBio;
          notify({ id: nid, text: "Updated Bio", type: "success" });
        }
      })
      .catch((err) => {
        notify({
          id: nid,
          text: err?.response?.data?.error ?? "Failed to update bio",
          type: "error"
        });
      });
  }

  function avatarDropped(ev: Event) {
    const files = (ev.currentTarget as HTMLInputElement)?.files;
    if (!files || files?.length <= 0) {
      console.error("avatarDropped: no file found");
      return;
    }
    const nid = notify({ text: "Uploading avatar", type: "loading" });
    axios
      .postForm(
        "/user/avatar",
        { avatar: files[0] },
        {
          headers: {
            "Content-Type": "multipart/form-data"
          }
        }
      )
      .then((r) => {
        if (user) {
          user.avatar = r.data as Image;
          notify({ id: nid, text: "Avatar Uploaded", type: "success" });
        }
      })
      .catch((err) => {
        console.error("uploading avatar failed", err);
        notify({
          id: nid,
          text: err?.response?.data?.error ?? "Failed to upload avatar",
          type: "error"
        });
      });
  }

  async function downloadWatchedList() {
    const nid = notify({ text: "Exporting", type: "loading" });
    try {
      exportDisabled = true;
      // We re-fetch, to ensure data we export is up to date.
      const r = await axios.get("/watched");
      console.log(r.data);
      if (!r.data || r.data?.length <= 0) {
        notify({
          id: nid,
          text: "Can't export an empty watch list!",
          type: "error",
          time: 10000
        });
        exportDisabled = false;
        return;
      }
      const file = new Blob([JSON.stringify(r.data, undefined, 2)], {
        type: "application/json"
      });
      const a = document.createElement("a");
      a.href = URL.createObjectURL(file);
      a.download = "watcharr-export.json";
      a.click();
      exportDisabled = false;
      notify({ id: nid, text: "Successfully Exported", type: "success" });
    } catch (err) {
      console.error("downloadWatchedList failed!", err);
      notify({ id: nid, text: "Export Failed!", type: "error" });
    }
  }

  /**
   * Takes in number of minutes and converts to readable.
   * eg into months, weeks, days, hours and minutes.
   */
  function toFormattedTimeLong(m: number) {
    // Considers a 30 days long month
    const countInMinutes = [
      ["month", 43200],
      ["week", 10080],
      ["day", 1440],
      ["hour", 60]
    ];

    let ansString = "";
    let tmp;
    for (const c of countInMinutes) {
      tmp = Math.floor(m / (c[1] as number));

      // Ignore fields with fewer than 1 unit
      if (tmp) ansString += `${tmp} ${c[0]}${tmp >= 2 ? "s, " : ", "}`;
      m -= tmp * (c[1] as number);
    }

    return ansString.slice(0, -2);
  }
</script>

<svelte:head>
  <title>My Profile</title>
</svelte:head>

<div class="content">
  <div class="inner">
    <div class="user-basic-info">
      <UserAvatar img={user?.avatar} {avatarDropped} />
      <div>
        <h2 title={user?.username}>
          <span style="font-weight: normal; font-variant: all-small-caps;">Hey</span>
          {user?.username}
        </h2>
        <textarea rows="1" placeholder="my bio" on:blur={updateBio} value={user?.bio} />
      </div>
    </div>

    <Stats>
      {#await getProfilePromise}
        <Spinner />
      {:then profile}
        <Stat name="Joined" value={formatDate(new Date(profile.joined))} />
        <Stat name="Movies Watched" value={profile.moviesWatched} large />
        <Stat name="Shows Watched" value={profile.showsWatched} large />
        <Stat name="Watching Movies" value={toFormattedTimeLong(profile.moviesWatchedRuntime)} />
        <Stat
          name="Watching Shows"
          value={toFormattedTimeLong(profile.showsWatchedRuntime)}
          disc="This is very inaccurate 🚀"
        />
      {:catch err}
        <Error error={err} pretty="Failed to get stats!" />
      {/await}
    </Stats>

    <div class="settings">
      <h3 class="norm">Settings</h3>

      <div class="theme">
        <h4 class="norm">Theme</h4>
        <div class="row">
          <button
            class={`plain${selectedTheme === "light" ? " selected" : ""}`}
            id="light"
            on:click={() => toggleTheme("light")}
          >
            light
          </button>
          <button
            class={`plain${selectedTheme === "dark" ? " selected" : ""}`}
            id="dark"
            on:click={() => toggleTheme("dark")}
          >
            dark
          </button>
        </div>
      </div>

      <Setting
        title="Country"
        desc="What country would you like to see available streaming providers for?"
      >
        <RegionDropDown
          selectedCountry={settings?.country}
          disabled={countryDisabled}
          onChange={(c) => {
            countryDisabled = true;
            updateUserSetting("country", c, () => {
              countryDisabled = false;
            });
          }}
        />
      </Setting>

      <Setting title="Private" desc="Hide your profile from others?" row>
        <Checkbox
          name="private"
          disabled={privateDisabled}
          value={settings?.private}
          toggled={(on) => {
            privateDisabled = true;
            updateUserSetting("private", on, () => {
              privateDisabled = false;
            });
          }}
        />
      </Setting>

      {#if !settings?.private}
        <Setting
          title="Private Thoughts"
          desc="Hide your watched list thoughts from followers?"
          row
        >
          <Checkbox
            name="privateThoughts"
            disabled={privateDisabled}
            value={settings?.privateThoughts}
            toggled={(on) => {
              privateThoughtsDisabled = true;
              updateUserSetting("privateThoughts", on, () => {
                privateThoughtsDisabled = false;
              });
            }}
          />
        </Setting>
      {/if}

      <Setting title="Hide Spoilers" desc="Do you want to hide episode info?" row>
        <Checkbox
          name="hideSpoilers"
          disabled={hideSpoilersDisabled}
          value={settings?.hideSpoilers}
          toggled={(on) => {
            hideSpoilersDisabled = true;
            updateUserSetting("hideSpoilers", on, () => {
              hideSpoilersDisabled = false;
            });
          }}
        />
      </Setting>

      <Setting
        title="Automate Show Statuses"
        desc="Do you want to automate show statuses (show, season, episode)?"
        tag="experimental"
        row
      >
        <Checkbox
          name="automateShowStatusesDisabled"
          disabled={automateShowStatusesDisabled}
          value={settings?.automateShowStatuses}
          toggled={(on) => {
            automateShowStatusesDisabled = true;
            updateUserSetting("automateShowStatuses", on, () => {
              automateShowStatusesDisabled = false;
            });
          }}
        />
      </Setting>

      <Setting
        title="Include Previously Watched"
        desc="Do you want to include previously watched content in the 'finished' filter, even if their status has now been changed?"
        row
      >
        <Checkbox
          name="includePreviouslyWatched"
          disabled={includePreviouslyWatchedDisabled}
          value={settings?.includePreviouslyWatched}
          toggled={(on) => {
            includePreviouslyWatchedDisabled = true;
            updateUserSetting("includePreviouslyWatched", on, () => {
              includePreviouslyWatchedDisabled = false;
              // Get profile stats again
              getProfilePromise = getProfile();
            });
          }}
        />
      </Setting>

      <RatingSetting />

      <div class="row btns">
        <button on:click={() => goto("/import")}>Import</button>
        <button on:click={() => downloadWatchedList()} disabled={exportDisabled}>Export</button>
        {#if user?.type !== UserType.Plex && user?.type !== UserType.Jellyfin}
          <button
            on:click={() => {
              pwChangeModalOpen = true;
            }}>Change Password</button
          >
        {/if}
        {#if user?.type === UserType?.Jellyfin}
          <button on:click={() => (jellyfinSyncModalOpen = true)} disabled={exportDisabled}>
            Sync With {localStorage.getItem("useEmby") ? "Emby" : "Jellyfin"}
          </button>
        {/if}
        {#if user?.type === UserType?.Plex}
          <button on:click={() => (plexSyncModalOpen = true)} disabled={exportDisabled}>
            Sync With Plex
          </button>
        {/if}
      </div>
      {#if pwChangeModalOpen}
        <PwChangeModal
          userName={user?.username}
          onClose={() => {
            pwChangeModalOpen = false;
          }}
        ></PwChangeModal>
      {/if}
      {#if jellyfinSyncModalOpen}
        <SyncModal onClose={() => (jellyfinSyncModalOpen = false)} />
      {/if}
      {#if plexSyncModalOpen}
        <SyncModal type="plex" onClose={() => (plexSyncModalOpen = false)} />
      {/if}
    </div>
  </div>
</div>

<style lang="scss">
  .content {
    display: flex;
    width: 100%;
    justify-content: center;
    padding: 0 30px 30px 30px;

    .inner {
      min-width: 400px;
      max-width: 400px;
      overflow: hidden;

      h2 {
        overflow: hidden;
        white-space: nowrap;
        text-overflow: ellipsis;
      }

      & > div:not(:first-of-type) {
        margin-top: 30px;
      }

      @media screen and (max-width: 440px) {
        width: 100%;
        min-width: unset;
      }
    }
  }

  .user-basic-info {
    display: flex;
    gap: 20px;

    & > div {
      display: flex;
      flex-flow: column;
      gap: 5px;
      width: 100%;
      overflow: hidden;

      textarea {
        resize: none;

        &:not(:focus) {
          border: 0;
          padding: 0;
          height: 32px;
        }
      }
    }
  }

  .settings {
    display: flex;
    flex-flow: column;
    gap: 20px;
    width: 100%;

    h3 {
      font-variant: small-caps;
    }

    & > div {
      margin: 0 15px;
    }

    div {
      &.row {
        display: flex;
        flex-flow: row;
        gap: 10px;
        align-items: center;

        &.btns button {
          width: max-content;
        }
      }
    }

    .theme {
      display: flex;
      flex-flow: column;
      gap: 10px;

      & button {
        width: 50%;
        height: 80px;
        border-radius: 10px;
        outline: 3px solid;
        font-size: 20px;
        text-transform: uppercase;
        font-family: "Rampart One";
        color: transparent;
        transition: all 200ms ease-in;

        &#light {
          background-color: white;
          outline-color: $accent-color;
          &:hover {
            color: black;
            -webkit-text-stroke: 0.5px black;
          }
        }

        &#dark {
          background-color: black;
          outline-color: white;
          &:hover {
            color: white;
            -webkit-text-stroke: 0.5px white;
          }
        }

        &.selected {
          outline-color: gold !important;
        }
      }
    }
  }
</style>
