<template>
<div class="row">
	<div v-bind:class="allClass">
		<div class="card card-secondary card-outline">
			<div class="card-body p-2">
				<dl class="row mb-0">
					<!-- metdata (default) -->
					<dt v-bind:class="dtClass">Create at</dt><dd v-bind:class="ddClass">{{ this.getTimestampString(metadata.creationTimestamp)}} ago ({{ metadata.creationTimestamp }})</dd>
					<dt v-bind:class="dtClass">Name</dt><dd v-bind:class="ddClass">{{ metadata.name }}</dd>
					<dt v-if="metadata.namespace" v-bind:class="dtClass">Namespace</dt><dd v-if="metadata.namespace" v-bind:class="ddClass">{{ metadata.namespace }}</dd>
					<dt v-if="metadata.labels && metadata.labels!={}" v-bind:class="dtClass">Labels</dt>
					<dd v-if="metadata.labels && metadata.labels!={}" v-bind:class="ddClass">
						<span v-for="(value, name) in metadata.labels" v-bind:key="name" class="label">{{ name }}={{ value }}</span>
					</dd>
					<dt v-if="metadata.annotations" v-bind:class="dtClass">Annotations</dt>
					<dd v-if="metadata.annotations" v-bind:class="ddClass">
						<a v-if="metadata.annotations" href="#"><b-icon :icon="isEllipseAnnotations?'arrows-expand':'arrows-collapse'" class="float-right mt-2" @click="isEllipseAnnotations=!isEllipseAnnotations"></b-icon></a>
						<ul class="list-unstyled mb-0">
							<li v-for="(v, k) in metadata.annotations" v-bind:key="k" v-bind:class="{'text-truncate':isEllipseAnnotations}">{{ k }}=<span class="font-weight-light">{{ v }}</span></li>
						</ul>
					</dd>
					<dt v-if="metadata.ownerReferences" v-bind:class="dtClass">Controlled By</dt>
					<dd v-if="metadata.ownerReferences" v-bind:class="ddClass">{{ controlledBy.kind }} <a href="#" @click="$emit('navigate', getViewLink(controlledBy.group, controlledBy.resource, metadata.namespace, controlledBy.name))">{{ controlledBy.name }}</a></dd>
					<!-- workload #1/2 (selector, nodeSelector, images) -->
					<dt v-if="Object.keys(workloadSpec.selector).length>0" v-bind:class="dtClass">Selector</dt>
					<dd v-if="Object.keys(workloadSpec.selector).length>0" v-bind:class="ddClass">
						<span v-for="(v, k) in workloadSpec.selector" v-bind:key="k" class="border-box background">{{k}}={{v}}</span>
					</dd>
					<dt v-if="Object.keys(workloadSpec.nodeSelector).length>0" v-bind:class="dtClass">Node Selector</dt>
					<dd v-if="Object.keys(workloadSpec.nodeSelector).length>0" v-bind:class="ddClass">
						<span v-for="(v, k) in workloadSpec.nodeSelector" v-bind:key="k" class="border-box background">{{k}}={{v}}</span>
					</dd>
					<dt v-if="workloadSpec.images.length>0" v-bind:class="dtClass">Images</dt>
					<dd v-if="workloadSpec.images.length>0" v-bind:class="ddClass">
						<ul class="list-unstyled mb-0">
							<li v-for="(v, i) in workloadSpec.images" v-bind:key="i">{{ v }}</li>
						</ul>
					</dd>
					<!-- slot -->
					<slot></slot>
					<!-- workload #2/2 (tolerations, affinity) -->
					<dt v-if="workloadSpec.tolerations.length>0" v-bind:class="dtClass">Tolerations</dt>
					<dd v-if="workloadSpec.tolerations.length>0" v-bind:class="ddClass">
						{{ workloadSpec.tolerations.length }}
						<a class="ml-2 text-sm" href="#" @click="toggle.tolerations=!toggle.tolerations">{{toggle.tolerations ? 'Hide' : 'Show'}}</a>
						<b-collapse  v-model="toggle.tolerations">
							<b-table-lite :items="workloadSpec.tolerations" class="subset"></b-table-lite>
						</b-collapse>
					</dd>
					<dt v-show="Object.keys(workloadSpec.affinity).length>0" v-bind:class="dtClass">Affinities</dt>
					<dd v-show="Object.keys(workloadSpec.affinity).length>0" v-bind:class="ddClass">
						<a href="#"><b-icon :icon="toggle.affinity?'arrows-expand':'arrows-collapse'" class="float-right mt-2" @click="toggle.affinity=!toggle.affinity"></b-icon></a>
						<p class="text-truncate font-weight-light mb-0">{{ workloadSpec.affinity }}</p>
						<b-collapse  v-model="toggle.affinity">
							<c-jsontree v-model="workloadSpec.affinity" class="card-body p-2 border"></c-jsontree>
						</b-collapse>
					</dd>
					<dt v-bind:class="dtClass">UID</dt><dd  v-bind:class="ddClass">{{ metadata.uid }}</dd>
				</dl>
			</div>
		</div>
	</div>
</div>
</template>
<script>
import VueJsonTree		from "@/components/jsontree";

export default {
	props:["dtCols","ddCols","size"],
	components: {
		"c-jsontree": { extends: VueJsonTree }
	},
	data () {
		return {
			metadata: {},
			allClass: `col-${this.size?this.size:'sm'}-12`,
			dtClass: `col-${this.size?this.size:'sm'}-${this.dtCols?this.dtCols:'2'}`,
			ddClass: `col-${this.size?this.size:'sm'}-${this.ddCols?this.ddCols:'10'}`,
			isEllipseAnnotations: true,
			controlledBy: {},
			summary: [],
			workloadSpec : {
				selector: {},
				nodeSelector: {},
				tolerations: [],
				affinity: {},
				images: []
			},
			toggle : {
				tolerations: false,
				affinity: false
			}
		}
	},
	mounted() {
		this.$nuxt.$on("view-data-read-completed", (data) => {
			if(!data) return
			this.metadata = data.metadata;
			this.controlledBy = data.metadata.ownerReferences?this.getResource(data.metadata.ownerReferences[0]):{};
			this.workloadSpec = { tolerations : [], affinity : {}, nodeSelector: {}, images: [], selector:{} };
			if (["Deployment","Job","Pod","ReplicaSet","StatefulSet"].includes(data.kind) && data.spec) {
				let spec = data.spec;
				if (spec.template && spec.template.spec) {
					this.workloadSpec = {
						tolerations : spec.template.spec.tolerations || [],
						affinity : spec.template.spec.affinity || {},
						nodeSelector: spec.template.spec.nodeSelector || {},
						images: spec.template.spec.containers?spec.template.spec.containers.map(d=>d.image):[]
					}
				} else if (!spec.template) {
					this.workloadSpec = {
						tolerations : spec.tolerations || [],
						affinity : spec.affinity || {},
						nodeSelector: spec.nodeSelector || {},
						images: spec.containers?spec.containers.map(d=>d.image):[]
					}
				}
				this.workloadSpec.selector =  spec.selector ? spec.selector.matchLabels: {}
			}


		});
	},
	beforeDestroy(){
		this.$nuxt.$off("view-data-read-completed");
	}
}
</script>
