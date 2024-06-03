<script lang="ts">
	import * as WebIFC from 'web-ifc';
	import { onMount } from 'svelte';

	interface PropertyMappings {
		Productcode: string;
		Modulenaam: string;
		Station: string;
		Aantal: string;
		Materiaal: string;
		Dikte: string;
		Gewicht: string;
		Volume: string;
		'Naa.K.T': string;
	}

	interface IfcElement {
		name: string;
		productcode: string;
		modulenaam: string;
		station: string;
		aantal: number;
		materiaal: string;
		dikte: string;
		gewicht: string;
		volume: string;
		code: string;
	}

	interface IfcType {
		typeID: number;
		typeName: string;
	}

	onMount(async () => {
		const ifcAPI = new WebIFC.IfcAPI();
		await ifcAPI.Init();
		const response = await fetch('/test.ifc');
		const fileContent = await response.arrayBuffer();
		const rawData = new Uint8Array(fileContent);
		const modelID = ifcAPI.OpenModel(rawData);

		const propertyMappings: PropertyMappings = {
			Productcode: 'productcode',
			Modulenaam: 'modulenaam',
			Station: 'station',
			Aantal: 'aantal',
			Materiaal: 'materiaal',
			Dikte: 'dikte',
			Gewicht: 'gewicht',
			Volume: 'volume',
			'Naa.K.T': 'code'
		};

		const elements: IfcElement[] = [];
		const allModelTypes: IfcType[] = ifcAPI.GetAllTypesOfModel(modelID);

		for (const type of allModelTypes) {
			if (ifcAPI.IsIfcElement(type.typeID)) {
				const elementIDs = ifcAPI.GetLineIDsWithType(modelID, type.typeID);
				for (let i = 0; i < elementIDs.size(); i++) {
					const elementID = ifcAPI.GetLine(modelID, elementIDs.get(i));
					const element: Partial<IfcElement> = { name: elementID.Name.value };

					const propSet = await ifcAPI.properties.getPropertySets(
						modelID,
						elementID.expressID,
						true
					);
					propSet.forEach((property) => {
						property.HasProperties.forEach((prop: any) => {
							const key = prop.Name.value as keyof PropertyMappings;
							if (propertyMappings[key]) {
								element[propertyMappings[key] as keyof IfcElement] = prop.NominalValue.value;
							}
						});
					});
					elements.push(element as IfcElement);
				}
			}
		}

		console.log(elements);
		console.log(elements.reduce((acc, elem) => acc + (elem.aantal || 0), 0));

		const combinedElements: IfcElement[] = Object.values(
			elements.reduce(
				(acc, element) => {
					const key = `${element.productcode}-${element.modulenaam}-${element.station}`;
					if (!acc[key]) {
						acc[key] = { ...element };
					} else {
						acc[key].aantal += element.aantal;
						acc[key].volume += element.volume;
						acc[key].gewicht += element.gewicht;
					}
					return acc;
				},
				{} as { [key: string]: IfcElement }
			)
		);

		console.log(combinedElements);
		console.log(combinedElements.reduce((acc, elem) => acc + (elem.aantal || 0), 0));
	});
</script>
