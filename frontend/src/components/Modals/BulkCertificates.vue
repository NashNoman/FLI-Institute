<template>
	<Dialog
		v-model="show"
		:options="{
			title: __('Generate Certificates'),
			size: 'lg',
			actions: [
				{
					label: __('Create'),
					variant: 'solid',
					onClick: ({ close }) => {
						generateCertificates(close)
					},
				},
			],
		}"
	>
		<template #body-content>
			<div class="space-y-4">
				<FormControl
					type="date"
					v-model="details.issue_date"
					:label="__('Issue Date')"
				/>
				<FormControl
					type="date"
					v-model="details.expiry_date"
					:label="__('Expiry Date')"
				/>
				<FormControl
					type="select"
					v-model="details.course"
					:label="__('Course')"
					:options="getCourses()"
				/>
				<Link
					v-model="details.template"
					:label="__('Certificate Template')"
					doctype="LMS Certificate Template"
					:filters="{ enabled: 1 }"
				/>
				<Switch
					size="sm"
					:label="__('Published')"
					:description="
						__(
							'Enabling this will publish the certificate on the certified participants page.',
						)
					"
					v-model="details.published"
				/>
				<div v-if="summary" class="space-y-2 text-sm border-t pt-3">
					<div class="text-ink-gray-9 font-medium">
						{{
							__('{0} certificates created').replace(
								'{0}',
								summary.created.length,
							)
						}}
					</div>
					<div v-if="summary.skipped.length">
						<div class="text-ink-amber-3 font-medium">
							{{ __('Skipped') }} ({{ summary.skipped.length }})
						</div>
						<ul class="list-disc pl-5 text-ink-gray-7">
							<li v-for="item in summary.skipped" :key="item.member">
								{{ item.member }}: {{ item.reason }}
							</li>
						</ul>
					</div>
					<div v-if="summary.failed.length">
						<div class="text-ink-red-3 font-medium">
							{{ __('Failed') }} ({{ summary.failed.length }})
						</div>
						<ul class="list-disc pl-5 text-ink-gray-7">
							<li v-for="item in summary.failed" :key="item.member">
								{{ item.member }}: {{ item.error }}
							</li>
						</ul>
					</div>
				</div>
			</div>
		</template>
	</Dialog>
</template>
<script setup>
import { inject, reactive, ref, watch } from 'vue'
import { call, Dialog, FormControl, Switch, toast } from 'frappe-ui'
import { cleanError } from '@/utils'
import Link from '@/components/Controls/Link.vue'

const show = defineModel()
const dayjs = inject('$dayjs')
const summary = ref(null)
const details = reactive({
	issue_date: dayjs().format('YYYY-MM-DD'),
	expiry_date: null,
	template: null,
	course: null,
	published: true,
})

const props = defineProps({
	batch: {
		type: [Object, null],
		required: true,
	},
})

const resolveTemplate = () => {
	call(
		'lms.lms.doctype.lms_certificate.lms_certificate.get_resolved_certificate_template',
		{
			course: details.course,
			batch: props.batch?.name,
		},
	)
		.then((template) => {
			details.template = template
		})
		.catch(() => {
			details.template = null
		})
}

const reset = () => {
	summary.value = null
	details.issue_date = dayjs().format('YYYY-MM-DD')
	details.expiry_date = null
	details.published = true
	details.course = props.batch?.courses?.[0]?.course || null
	resolveTemplate()
}

watch(show, (value) => {
	if (value) reset()
})

watch(
	() => details.course,
	() => {
		if (show.value) resolveTemplate()
	},
)

const generateCertificates = (close) => {
	call(
		'lms.lms.doctype.lms_certificate.lms_certificate.create_bulk_certificates',
		{
			batch: props.batch.name,
			template: details.template,
			issue_date: details.issue_date,
			expiry_date: details.expiry_date,
			published: details.published ? 1 : 0,
			course: details.course,
		},
	)
		.then((result) => {
			summary.value = result
			if (!result.skipped.length && !result.failed.length) {
				toast.success(__('Certificates generated successfully'))
				close()
			} else if (result.failed.length) {
				toast.warning(
					__('Some certificates could not be generated. See the details.'),
				)
			} else {
				toast.info(__('Some students were skipped. See the details.'))
			}
		})
		.catch((err) => {
			toast.error(cleanError(err.messages?.[0]) || err.message || err)
		})
}

const getCourses = () => {
	return props.batch?.courses.map((course) => {
		return {
			label: course.course,
			value: course.course,
		}
	})
}
</script>
