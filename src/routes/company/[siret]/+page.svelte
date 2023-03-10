<!--suppress JSDeprecatedSymbols, JSDeprecatedSymbols -->
<script lang="ts">
    import {page} from "$app/stores";
    import {onMount} from "svelte";
    import {
        Accordion,
        AccordionItem,
        Alert,
        Button,
        Card,
        DarkMode,
        Heading,
        Indicator,
        Kbd,
        P,
        Popover,
        Spinner
    } from 'flowbite-svelte';
    import {browser, dev} from "$app/environment";
    import {type Report, uuid_e4} from "$libs/public-api";
    import {type FirebaseApp, initializeApp} from "firebase/app";
    import {type Analytics, getAnalytics} from "@firebase/analytics";
    import {type FirebasePerformance, getPerformance} from "@firebase/performance";
    import {environment} from "../../../environments/environment";
    import {type Database, type DataSnapshot, getDatabase, onValue, ref, set} from "@firebase/database";
    import {type Company} from "french-company-types";
    import {FooterComponent, InfoSvg, TableLeaders, TableLogs} from "$components/public-api";
    import Tab1 from "./Tab1.svelte";
    import {slide} from "svelte/transition";
    import {type AppCheck, initializeAppCheck, ReCaptchaV3Provider} from "@firebase/app-check";

    let firebaseApp: FirebaseApp | undefined
    let firebaseAppCheck: AppCheck | undefined
    let firebaseAnalytics: Analytics | undefined
    let firebasePerformance: FirebasePerformance | undefined
    let firebaseDatabase: Database | undefined

    /** @type {import('./$types').PageServerData} */
    export let data

    const scoreLogs: any[] = []
    let score: number | undefined
    let maxScore: number | undefined

    let isError: boolean = false
    let isFetch: boolean = false

    const siret: string | undefined = $page.params.siret

    let company: Company | undefined = undefined
    let companyReportCount: number = 0

    let currentTab = 0

    onMount(async () => {

        if (dev) self["FIREBASE_APPCHECK_DEBUG_TOKEN"] = true

        if (browser) {
            firebaseApp = initializeApp(environment.firebaseConfig)
            firebaseAppCheck = initializeAppCheck(firebaseApp, {
                provider: new ReCaptchaV3Provider('6LeN3u8kAAAAAMqcFHooMnaEGk2j_MNAZpUQFD_X'),
                isTokenAutoRefreshEnabled: true
            })
            firebaseAnalytics = getAnalytics(firebaseApp)
            firebasePerformance = getPerformance(firebaseApp)
            firebaseDatabase = getDatabase(firebaseApp)
        }

        if (!siret) return isFetch = true
        try {
            const response: Response = await fetch(encodeURI(`${$page.url.origin}/api/company?q=${siret}`))
            const _temps = await response.json()
            if (!_temps || _temps.length < 1) return isFetch = true
            company = _temps[0]
            isFetch = true

            const _startTime = new Date().getTime()
            const testValue = [
                {label: 'Raison sociale', value: company?.nom_raison_sociale},
                {label: 'Nom complet', value: company?.nom_complet},
                {label: 'Siren', value: company?.siren},
                {label: 'Nature juridique', value: company?.nature_juridique},
                {label: 'Libell?? nature juridique N1', value: company?.libelle_nature_juridique_n1},
                {label: 'Libell?? nature juridique N2', value: company?.libelle_nature_juridique_n2},
                {label: 'Libell?? nature juridique N3', value: company?.libelle_nature_juridique_n3},
                {label: 'Activit?? principale', value: company?.activite_principale},
                {label: 'Libell?? activit?? principale N1', value: company?.libelle_activite_principale_n1},
                {label: 'Libell?? activit?? principale N2', value: company?.libelle_activite_principale_n2},
                {label: 'Libell?? activit?? principale N3', value: company?.libelle_activite_principale_n3},
                {label: 'Libell?? activit?? principale N4', value: company?.libelle_activite_principale_n4},
                {label: 'Libell?? activit?? principale N5', value: company?.libelle_activite_principale_n5},
                {label: 'Nombre d\'??tablissements', value: company?.nombre_etablissements},
                {label: 'Nombre d\'??tablissements ouverts', value: company?.nombre_etablissements_ouverts},
                {label: 'Tranche effectif salari??', value: company?.tranche_effectif_salarie},
                {label: 'Cat??gorie d\'entreprise', value: company?.categorie_entreprise},
                {label: '??tat administratif', value: company?.etat_administratif},
                {label: 'Dirigeants', value: company?.dirigeants},
                {label: '??tablissements li??', value: company?.matching_etablisssements},
            ]
            score = testValue.length * 10
            maxScore = testValue.length * 10

            for (let testValueElement of testValue) {
                if (!testValueElement.value) score -= 10
                scoreLogs.push({
                    initialScoreValue: score + (!testValueElement.value ? 10 : 0),
                    scoreValue: score,
                    label: testValueElement.label,
                    at: Math.abs(_startTime - new Date().getTime())
                })
            }

            score += 100
            scoreLogs.push({
                initialScoreValue: score - 100,
                scoreValue: score,
                label: 'Ratio Nombre d\'??tablissements',
                at: Math.abs(_startTime - new Date().getTime())
            })
            maxScore += 100
            if (company && company.nombre_etablissements && company.nombre_etablissements_ouverts) {
                const _t = 100 - (((company.nombre_etablissements - company.nombre_etablissements_ouverts) / company.nombre_etablissements * 100).toFixed(0))
                score -= _t
                scoreLogs.push({
                    initialScoreValue: score + _t,
                    scoreValue: score,
                    label: 'Ratio Nombre d\'??tablissements',
                    at: Math.abs(_startTime - new Date().getTime())
                })
            } else {
                score -= 100
                scoreLogs.push({
                    initialScoreValue: score + 100,
                    scoreValue: score,
                    label: 'Ratio Nombre d\'??tablissements',
                    at: Math.abs(_startTime - new Date().getTime())
                })
            }

            try {
                if (siret && firebaseDatabase) {
                    onValue(ref(firebaseDatabase, `reports/${siret}`), (dataSnapshot: DataSnapshot) => {
                        let data: Record<string, Report> | undefined | null
                        if (dataSnapshot.exists() && (data = dataSnapshot.val()) && data && Object.values(data) && Object.values(data).length) {
                            companyReportCount = Object.values(data).length
                        }
                    })
                }
            } catch (reason) {
                if (dev) console.log(reason)
            }

        } catch (reason) {
            if (dev) console.log(reason)
            isError = true
        }
    })

    const reportCompany = async () => {
        if (!siret || !firebaseApp || !firebaseDatabase) return
        try {
            await set(ref(firebaseDatabase, `reports/${siret}/${uuid_e4()}`), {
                clientInformation: {
                    appCodeName: navigator.appCodeName,
                    appName: navigator.appName,
                    appVersion: navigator.appVersion,
                    platform: navigator.platform,
                    userAgent: navigator.userAgent,
                    languages: navigator.languages,
                    cookieEnabled: navigator.cookieEnabled,
                    geolocation: navigator.geolocation,
                    doNotTrack: navigator.doNotTrack,
                    history: window.history,
                    caches: window.caches,
                },
                clientIp: data.clientIp,
                at: new Date(Date.now()).toISOString()
            })
        } catch (reason) {
            if (dev) console.log(reason)
        }
    }

    const getScoreRatio = () => {
        if (!score || !maxScore) return 0
        return (score / maxScore) * 100
    }

</script>

<svelte:head>
    <title>NuStateAPI | Company | {siret}</title>
</svelte:head>

{#if !siret || !company || !isFetch}
    {#if !isFetch}
        <section class="absolute top-0 right-0 left-0 bottom-0 flex items-center justify-center">
            <article class="relative">
                <Spinner class="drop-shadow-md" size={12}></Spinner>
            </article>
        </section>
    {:else}
        {#if isError}
            <section class="mx-auto w-1/2 my-6 drop-shadow-md">
                <article>
                    <Alert class="drop-shadow-md" color="red">
                        <span slot="icon"><InfoSvg></InfoSvg></span>
                        <span class="font-medium">Erreur</span> interne survenue.
                    </Alert>
                </article>
            </section>
        {:else}
            <section class="mx-auto w-1/2 my-6 drop-shadow-md">
                <article>
                    <Alert class="drop-shadow-md" color="red">
                        <span slot="icon"><InfoSvg></InfoSvg></span>
                        <span class="font-medium">Num??ro de SIRET</span> invalide.
                    </Alert>
                </article>
            </section>
        {/if}
    {/if}
{:else if isFetch && company}
    <section class="relative mx-6 lg:mx-24 mt-14 mb-3 flex flex-col-reverse lg:flex-row lg:space-x-3
                    transition-all duration-300 ease-in-out">
        <article class="max-lg:mt-3 lg:!w-[calc(50%-1rem)] h-fit shadow-inner rounded-lg">
            <Card class="max-lg:mt-3 !bg-transparent text-left items-start"
                  size="full">
                <Heading class="flex inline-flex drop-shadow-md" tag="h1">
                    {company?.nom_raison_sociale ?? 'N/A'}
                    <Indicator size="lg" color={company?.nom_raison_sociale ? 'green' : 'red'}/>
                </Heading>
                <Heading class="flex inline-flex drop-shadow-md !text-lg !text-gray-500" tag="h2">
                    {company?.nom_complet ?? 'N/A'}
                    <Indicator size="sm" color={company?.nom_complet ? 'green' : 'red'}/>
                </Heading>

                <div class="my-3 mb-4 w-full flex inline-flex max-sm:space-x-1.5 sm:space-x-3">
                    <Button on:click={() => currentTab = 0} color={currentTab === 0 ? 'blue' : 'alternative'}
                            class="w-full max-sm:text-xs shadow drop-shadow-md transition-all duration-300 ease-in-out">
                        Entreprise
                    </Button>
                    <span class="w-full" id="btnTab2">
                        <Button disabled on:click={() => currentTab = 1}
                                color={currentTab === 1 ? 'blue' : 'alternative'}
                                class="w-full max-sm:text-xs shadow drop-shadow-md duration-300 ease-in-out">
                            Si??ge
                        </Button>
                    </span>
                    <Popover class="w-64 text-sm font-light" placement="top" title="Bient??t" transition={slide}
                             trigger="hover" triggeredBy="#btnTab2">
                        Prochainement les informations sur le si??ge.
                    </Popover>

                    <span class="w-full" id="btnTab3">
                        <Button disabled on:click={() => currentTab = 2}
                                color={currentTab === 2 ? 'blue' : 'alternative'}
                                class="w-full max-sm:text-xs shadow drop-shadow-md duration-300 ease-in-out">
                            Compl??ments
                        </Button>
                    </span>
                    <Popover class="w-64 text-sm font-light" placement="top" title="Bient??t" transition={slide}
                             trigger="hover" triggeredBy="#btnTab3">
                        Prochainement les informations compl??mentaires.
                    </Popover>
                </div>

                {#if currentTab === 0}
                    <Tab1 company={company}></Tab1>
                {/if}
                {#if currentTab === 1}

                {/if}
                {#if currentTab === 2}
                {/if}
            </Card>
        </article>
        <article class="lg:!w-[calc(50%-1rem)] flex flex-col space-y-3">
            <section class="shadow-md rounded-lg">
                <article>
                    <Card class="!bg-transparent text-left items-start gap-y-3 shadow-inner" size="full">
                        <Heading class="drop-shadow-md" tag="h1">SCORE</Heading>
                        <Heading class="drop-shadow-md" tag="h2">
                            <span class="{getScoreRatio() <  75 ? getScoreRatio() <  50 ? getScoreRatio() <  25 ? 'text-red-500' : 'text-orange-500' : 'text-yellow-500' : 'text-green-500'}">
                                {score ?? 'N/A'}
                            </span>
                            <span class="text-gray-400 dark:text-gray-500">
                                /{maxScore ?? 'N/A'} - {getScoreRatio()?.toFixed(2)}%
                            </span>
                        </Heading>
                    </Card>
                </article>
            </section>
            <section class="shadow-md rounded-lg">
                <article>
                    <Card shadow class="!bg-transparent text-left items-start gap-y-3 shadow-inner" size="full">
                        <Heading class="flex inline-flex justify-between drop-shadow-md" tag="h1">
                            Rapport
                            {#if firebaseApp && firebaseDatabase}
                                <Button on:click={reportCompany} color="red"
                                        class="transition-all duration-300 ease-in-out drop-shadow-md shadow dark:hover:opacity-75">
                                    Signaler
                                </Button>
                            {/if}
                        </Heading>
                        <Heading tag="h2"
                                 class="drop-shadow-md {companyReportCount <  75 ? companyReportCount <  50 ? companyReportCount <  25 ? '!text-green-500' : '!text-yellow-500': '!text-orange-500' : '!text-red-500' }">
                            {companyReportCount}
                        </Heading>
                    </Card>
                </article>
            </section>
        </article>
    </section>
    <section class="relative shadow-md rounded-lg mx-6 lg:mx-12 mb-3 transition-all duration-300 ease-in-out">
        <article>
            <Card shadow class="!bg-transparent text-left items-start gap-y-4 shadow-inner" size="full">
                <Heading class="drop-shadow-md" tag="h3">Logs</Heading>
                <div class="italic drop-shadow-md">
                    <P color="gray"><span class="font-bold">IS</span> : Valeur initial du score</P>
                    <P color="gray"><span class="font-bold">AS</span> : Valeur apr??s le test du score</P>
                </div>
                <Accordion
                        class="relative w-full transition-all duration-300 ease-in-out">
                    <AccordionItem>
                        <div slot="header">D??tails</div>
                        <TableLogs scoreLogs={scoreLogs}></TableLogs>
                    </AccordionItem>
                </Accordion>
            </Card>
        </article>
    </section>
    <section class="relative mx-6 lg:mx-12 mb-3 transition-all duration-300 ease-in-out">
        <article>
            <Card shadow class="!bg-transparent text-left items-start gap-y-4 shadow-inner" size="full">
                <Heading class="drop-shadow-md" tag="h3">
                    Dirigeant{company.dirigeants.length > 1 ? 's' : '(e)'}</Heading>
                <div class="italic drop-shadow-md">
                    <P color="gray"><span class="font-bold">PP</span> : Personne Physique</P>
                    <P color="gray"><span class="font-bold">PM</span> : Personne Morale</P>
                </div>
                <Accordion
                        class="relative w-full transition-all duration-300 ease-in-out">
                    {#if company?.dirigeants && company.dirigeants.length > 0}
                        <AccordionItem>
                            <div slot="header">D??tails sur {company.dirigeants.length > 1 ? 'les' : 'le/la'}
                                dirigeant{company.dirigeants.length > 1 ? 's' : '(e)'}</div>
                            <TableLeaders leaders={company?.dirigeants}></TableLeaders>
                        </AccordionItem>
                    {:else}
                        <AccordionItem class>
                            <div slot="header">Aucune information sur la/le(s) dirigeant(e/s)</div>
                            <Alert shadow color="red">
                                <span slot="icon"><InfoSvg className="h-4 w-4"></InfoSvg></span>
                                <p class="text-start">
                                    <span class="-ml-1 font-medium">Dirigeant(s) inconnu(e/s)</span>, le <Kbd
                                        class="px-2 py-1 max-sm:mt-1">score</Kbd> de l'entreprise est affect??.
                                </p>
                            </Alert>
                        </AccordionItem>
                    {/if}
                </Accordion>
            </Card>
        </article>
    </section>
    <section class="relative mx-6 lg:mx-12 mb-3 transition-all duration-300 ease-in-out">
        <article>
            <Card shadow class="!bg-transparent text-left items-start gap-y-4 shadow-inner" size="full">
                <Heading class="drop-shadow-md" tag="h3">??tablissement(s) li??</Heading>
                <!--div class="italic drop-shadow-md">
                    <P color="gray"><span class="font-bold">PP</span> : Personne Physique</P>
                    <P color="gray"><span class="font-bold">PM</span> : Personne Morale</P>
                </div-->
                <Accordion
                        class="relative w-full transition-all duration-300 ease-in-out">
                    {#if company?.matching_etablisssements && company.matching_etablisssements.length > 0}
                        <AccordionItem>
                            <div slot="header">D??tails sur {company.matching_etablisssements.length > 1 ? 'les' : 'l\''}
                                ??tablissement{company.matching_etablisssements.length > 1 ? 's' : '(e)'} li??
                            </div>
                            <!--TODO TableLeaders leaders={company?.dirigeants}></TableLeaders-->
                        </AccordionItem>
                    {:else}
                        <AccordionItem class>
                            <div slot="header">Aucune information sur un/des ??tablissement(s) li??</div>
                            <Alert shadow color="red">
                                <span slot="icon"><InfoSvg className="h-4 w-4"></InfoSvg></span>
                                <p class="text-start">
                                    <span class="-ml-1 font-medium">??tablissement(s) li?? inconnu(s)</span>, le <Kbd
                                        class="px-2 py-1 max-sm:mt-1">score</Kbd> de l'entreprise est affect??.
                                </p>
                            </Alert>
                        </AccordionItem>
                    {/if}
                </Accordion>
            </Card>
        </article>
    </section>
{/if}

{#if isFetch}
    <FooterComponent></FooterComponent>
{/if}

<div class="absolute top-0 right-0 m-2">
    <DarkMode></DarkMode>
</div>
