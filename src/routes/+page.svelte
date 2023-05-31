<script lang="ts">
	import { FileDropzone } from '@skeletonlabs/skeleton';
	import { XMLParser } from 'fast-xml-parser';

	let files: FileList;

	interface Transaction {
		date: string | null;
		amount: number;
		id: string | number;
		payee_name?: string | null;
		description?: string | null;
		tags?: string | null;
	}

	// Extracts the YYYY-MM-DD from the following formats:
	/// * YYYYMMDDHHMMSS.XXX[gmt offset:tz name] (official OFX format)
	/// * 20190123120000
	/// If an OFX/QFX file is seen with a different format, it should
	/// be supported by this function, even if it is not spec-compliant.
	function ofxDateParse(raw: string) {
		const longRx = /^(\d{4})(\d{2})(\d{2})(\d{2})(\d{2})(\d{2})(\.\d{3})?\[(.+):(.+)\]$/;
		const shortRx = /^(\d{4})(\d{2})(\d{2})(\d{2})(\d{2})(\d{2})$/;
		let res = longRx.exec(raw);
		if (res) {
			const [_, y, m, day, _hr, _min, _sec, _milli, _offsetStr, _tz] = res;
			return `${y}-${m}-${day}`;
		} else {
			res = shortRx.exec(raw);
			if (res) {
				const [_, y, m, day, _hr, _min, _sec] = res;
				return `${y}-${m}-${day}`;
			}
		}
		console.warn('Not able to parse date', raw);
		return null;
	}

	const parser = new XMLParser();
	const parseFile = async (file: File) => {
		const xmlData = await file.text();
		const data = parser.parse(xmlData);
		console.log(data);
		const bankTransactions = [
			(data.OFX.BANKMSGSRSV1?.STMTTRNRS.STMTRS || data.OFX.CREDITCARDMSGSRSV1?.CCSTMTTRNRS.CCSTMTRS)
				?.BANKTRANLIST.STMTTRN
		].flat();

		const invstTransactions = [
			data.OFX.INVSTMTMSGSRSV1?.INVSTMTTRNRS?.INVSTMTRS?.INVTRANLIST?.INVBANKTRAN
		].flat();

		const transactions: Transaction[] = [...bankTransactions, ...invstTransactions]
			.filter((trans) => trans)
			.map((trans) => {
				trans = trans?.STMTTRN || trans;
				return {
					amount: trans.TRNAMT,
					id: trans.FITID,
					date: trans.DTPOSTED ? ofxDateParse(trans.DTPOSTED + '') : null,
					payee_name: trans.NAME,
					description: trans.MEMO,
					tags: `${trans.TRNTYPE}`
				} satisfies Transaction;
			});

		return transactions;
	};

	const buildCSV = (transactions: Transaction[]) => {
		const header = 'Date,Amount,Description,ID,Payee';
		return [
			header,
			...transactions.map((data) =>
				[data.date, data.amount, data.description, data.id, data.payee_name].join(',')
			)
		].join('\n');
	};

	const download = async () => {
		const file = files?.item(0);
		if (!file) return;
		const csv = buildCSV(await parseFile(file));

		const csvFile = new Blob([csv]);

		const url = window.URL || window.webkitURL;
		const link = url.createObjectURL(csvFile);

		// generate anchor tag, click it for download and then remove it again
		const a = document.createElement('a');
		a.setAttribute('download', `${file.name}.csv`);
		a.setAttribute('href', link);
		document.body.appendChild(a);
		a.click();
		document.body.removeChild(a);
	};
</script>

<div class="container h-full mx-auto flex flex-col justify-center items-center">
	<FileDropzone name="ofx-file" bind:files accept=".ofx,.qfx" />
	{#if files?.length === 1}
		<button class="btn variant-filled-primary" on:click={download}>Download CSV</button>
	{/if}
</div>
