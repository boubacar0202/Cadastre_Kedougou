<script setup>
import AuthenticatedLayout from '@/Layouts/AuthenticatedLayout.vue';
import { Head, Link, usePage } from '@inertiajs/vue3';
import { computed } from 'vue';

const page = usePage();

const nombre = (cle) => Number(page.props[cle] ?? 0);
const format = (cle) => new Intl.NumberFormat('fr-FR').format(nombre(cle));

const dossiers = computed(() => [
    { libelle: 'Délimitations', cle: 'totalDelimitation', route: 'geometre.create' },
    { libelle: 'Avec NICAD', cle: 'totalNicad', route: 'matriceCadastrale.create' },
    { libelle: 'Sans NICAD', cle: 'totalSansNicad', route: 'donnee.create' },
]);

const courriers = computed(() => [
    { libelle: 'Arrivées', cle: 'totalCourriersArrivees', route: 'arrivee.create' },
    { libelle: 'Départs', cle: 'totalCourriersDeparts', route: 'depart.create' },
    { libelle: 'Instances', cle: 'totalInstances', route: 'instance.create' },
]);

const fond = "https://upload.wikimedia.org/wikipedia/commons/9/93/Map_of_the_departments_of_the_K%C3%A9dougou_region_of_Senegal.png";
</script>

<template>
    <Head title="Accueil">
        <link rel="icon" sizes="512x512" href="/logo-01.png">
    </Head>

    <AuthenticatedLayout>
        <template #header>
            <h2 class="text-xl font-semibold leading-tight text-primary-txt">
                Accueil
            </h2>
        </template>

        <div
            class="-m-4 min-h-[calc(100vh-4.5rem)] bg-cover bg-center bg-no-repeat px-4 py-6 sm:px-6 md:-my-6 md:-mr-6 md:ml-0 md:min-h-[calc(100vh-5rem)] md:py-8 md:pr-6"
            :style="{ backgroundImage: `url('${fond}')` }"
        >
            <p class="sr-only">Carte des départements de la région de Kédougou.</p>
            <div class="mx-auto w-full max-w-5xl">
            <section class="overflow-hidden rounded-2xl border border-white/80 bg-white/85 shadow-[0_18px_40px_-14px_rgba(97,49,4,0.35),inset_0_1px_0_rgba(255,255,255,0.9)] backdrop-blur-xl backdrop-saturate-150 supports-[backdrop-filter]:bg-white/55">
                <div class="flex flex-col gap-6 px-5 py-6 sm:px-7 sm:py-7 lg:flex-row lg:items-end lg:justify-between">
                    <div class="max-w-md">
                        <h3 class="text-xl font-bold tracking-tight text-primary">Dossiers</h3>
                        <p class="mt-2 text-sm leading-relaxed text-primary-txt">
                            Terrains enregistrés au centre, avec ou sans NICAD.
                        </p>
                        <Link
                            :href="route('secretariat.create')"
                            class="mt-5 inline-flex min-h-11 items-center rounded-xl bg-primary px-4 py-2.5 text-sm font-bold text-white shadow-[0_8px_16px_-6px_rgba(97,49,4,0.45)] hover:bg-primary-dark focus-visible:outline focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-[#613104]"
                        >
                            Enregistrer un dossier
                        </Link>
                    </div>
                    <p class="text-6xl font-bold leading-none tracking-tight text-primary tabular-nums">
                        {{ format('totalTerrains') }}
                        <span class="mt-2 block text-sm font-semibold tracking-normal text-primary-txt">dossiers</span>
                    </p>
                </div>
                <dl class="grid border-t border-white/70 bg-white/35 sm:grid-cols-3">
                    <div
                        v-for="(item, index) in dossiers"
                        :key="item.cle"
                        class="border-white/70 px-5 py-4"
                        :class="index > 0 ? 'border-t sm:border-l sm:border-t-0' : ''"
                    >
                        <dt class="text-sm font-semibold">
                            <Link
                                :href="route(item.route)"
                                class="text-primary underline-offset-4 hover:underline focus-visible:outline focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-[#613104]"
                            >
                                {{ item.libelle }}
                            </Link>
                        </dt>
                        <dd class="mt-1 text-2xl font-bold tabular-nums text-primary">{{ format(item.cle) }}</dd>
                    </div>
                </dl>
            </section>

            <section class="mt-10 overflow-hidden rounded-2xl border border-white/80 bg-white/85 shadow-[0_18px_40px_-14px_rgba(97,49,4,0.35),inset_0_1px_0_rgba(255,255,255,0.9)] backdrop-blur-xl backdrop-saturate-150 supports-[backdrop-filter]:bg-white/55">
                <div class="px-5 pt-5">
                    <h3 class="text-xl font-bold tracking-tight text-primary">Courrier</h3>
                    <p class="mt-2 text-sm leading-relaxed text-primary-txt">Arrivées, départs et courriers encore en instance.</p>
                </div>
                <ul class="mt-4 border-t border-white/70 bg-white/35">
                    <li
                        v-for="item in courriers"
                        :key="item.cle"
                        class="border-t border-white/70 first:border-t-0"
                    >
                        <Link
                            :href="route(item.route)"
                            class="flex min-h-14 items-center justify-between gap-4 px-5 py-4 transition-colors hover:bg-white/45 focus-visible:outline focus-visible:outline-2 focus-visible:outline-offset-[-2px] focus-visible:outline-[#613104]"
                        >
                            <span class="font-semibold text-primary">{{ item.libelle }}</span>
                            <span class="text-2xl font-bold tabular-nums text-primary">{{ format(item.cle) }}</span>
                        </Link>
                    </li>
                </ul>
            </section>
            </div>
        </div>
    </AuthenticatedLayout>
</template>
