<script>
    import he from 'he'

    import HollowButton from '../components/HollowButton.svelte'
    import { _ } from '../i18n'

    export let notifications
    export let eventTag = () => {}
    export let handlePlanAdd = () => {}

    const allowJiraImport = appConfig.AllowJiraImport

    function uploadFile() {
        let file = this.files[0]
        if (!file) {
            return
        }
        if (file.type !== 'text/xml') {
            notifications.danger($_('actions.plan.importJiraXML.badFileType'))
            eventTag('jira_import_failed', 'battle', `file.type not text/xml`)
            return
        }

        let reader = new FileReader()

        reader.readAsText(file)

        reader.onload = () => {
            try {
                const docParser = new DOMParser()
                const domContent = reader.result.replace(/<!--.*?-->/gis, '')
                const doc = docParser.parseFromString(
                    domContent,
                    'application/xml',
                )
                const items = doc.querySelectorAll('channel>item')
                if (items) {
                    const totalItems = items.length
                    for (let i = 0; i < totalItems; i++) {
                        const item = items[i]
                        const decodedDescription = he.decode(
                            item.querySelector('description').innerHTML,
                        )
                        const customFields = item.querySelectorAll(
                            'customfields>customfield',
                        )
                        let acceptanceCriteria = ''

                        if (customFields) {
                            for (let j = 0; j < customFields.length; j++) {
                                const cfName = customFields[j].querySelector(
                                    'customfieldname',
                                ).innerHTML
                                const cfValues = customFields[j].querySelector(
                                    'customfieldvalues',
                                ).innerHTML

                                if (
                                    cfName.toLowerCase() ===
                                    'acceptance criteria'
                                ) {
                                    acceptanceCriteria = cfValues
                                }
                            }
                        }

                        const plan = {
                            id: '',
                            planName: item.querySelector('summary').innerHTML,
                            type: item
                                .querySelector('type')
                                .innerHTML.toLowerCase(),
                            referenceId: item.querySelector('key').innerHTML,
                            link: item.querySelector('link').innerHTML,
                            description: decodedDescription,
                            acceptanceCriteria,
                        }
                        handlePlanAdd(plan)
                    }
                    eventTag(
                        'jira_import_success',
                        'battle',
                        `total stories imported: ${totalItems}`,
                    )
                }
            } catch (e) {
                notifications.danger(
                    $_('actions.plan.importJiraXML.errorReadingFile'),
                )
                eventTag('jira_import_failed', 'battle', `ferror reading file`)
            }
        }

        reader.onerror = () => {
            notifications.danger(
                $_('actions.plan.importJiraXML.errorReadingFile'),
            )
            eventTag('jira_import_failed', 'battle', `ferror reading file`)
        }
    }
</script>

{#if allowJiraImport}
    <HollowButton type="label" additionalClasses="mr-2" color="blue">
        {$_('actions.plan.importJiraXML.button')}
        <input type="file" on:change="{uploadFile}" class="hidden" />
    </HollowButton>
{/if}
